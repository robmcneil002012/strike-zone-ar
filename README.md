import React, { useState, useRef, useEffect, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
  Pressable,
  Dimensions,
  Platform,
} from 'react-native';
import { CameraView, useCameraPermissions } from 'expo-camera';
import { GLView } from 'expo-gl';
import { Renderer } from 'expo-three';
import * as THREE from 'three';
import { Magnetometer, DeviceMotion } from 'expo-sensors';
import { StatusBar } from 'expo-status-bar';

const { width: SCREEN_W, height: SCREEN_H } = Dimensions.get('window');

const MODES = {
  structure: { label: 'STRUCTURE', color: 0x00eaff, hex: '#00eaff' },
  dropoff: { label: 'DEPTH DROP', color: 0xffee00, hex: '#ffee00' },
  casting: { label: 'CAST TARGET', color: 0x39ff14, hex: '#39ff14' },
};

const GROUND_Y = -1.2;

function compassLabel(deg) {
  const dirs = ['N', 'NE', 'E', 'SE', 'S', 'SW', 'W', 'NW'];
  const idx = Math.round(((deg % 360) / 45)) % 8;
  return dirs[idx];
}

export default function App() {
  const [permission, requestPermission] = useCameraPermissions();
  const [targetMode, setTargetMode] = useState('structure');
  const [markers, setMarkers] = useState([]);
  const [hudVisible, setHudVisible] = useState(true);
  const [heading, setHeading] = useState(0);
  const [lastRange, setLastRange] = useState(null);
  const [targetStatus, setTargetStatus] = useState('SCANNING');

  const sceneRef = useRef(null);
  const cameraRef = useRef(null);
  const rendererRef = useRef(null);
  const glRef = useRef(null);
  const markersGroupRef = useRef(null);
  const animFrame = useRef(null);
  const deviceTiltRef = useRef({ x: 0, y: 0 });
  const lockTimeout = useRef(null);

  useEffect(() => {
    let magSub;
    let motionSub;
    try {
      Magnetometer.setUpdateInterval(500);
      magSub = Magnetometer.addListener((data) => {
        let angle = Math.atan2(data.y, data.x) * (180 / Math.PI);
        angle = (angle + 360) % 360;
        setHeading(Math.round(angle));
      });
    } catch (e) {}

    try {
      DeviceMotion.setUpdateInterval(100);
      motionSub = DeviceMotion.addListener((data) => {
        const rotation = data.rotation;
        if (!rotation) return;
        const beta = rotation.beta || 0;
        const gamma = rotation.gamma || 0;
        deviceTiltRef.current = {
          x: Math.max(-0.5, Math.min(0.5, beta / 3)),
          y: Math.max(-0.5, Math.min(0.5, gamma / 3)),
        };
      });
    } catch (e) {}

    return () => {
      magSub && magSub.remove();
      motionSub && motionSub.remove();
      if (animFrame.current) cancelAnimationFrame(animFrame.current);
      if (lockTimeout.current) clearTimeout(lockTimeout.current);
    };
  }, []);

  const buildMarkerMesh = useCallback((point, mode) => {
    const conf = MODES[mode];
    const group = new THREE.Group();

    const coneGeo = new THREE.ConeGeometry(0.12, 0.35, 16);
    const coneMat = new THREE.MeshStandardMaterial({
      color: conf.color,
      emissive: conf.color,
      emissiveIntensity: 0.6,
      transparent: true,
      opacity: 0.9,
    });
    const cone = new THREE.Mesh(coneGeo, coneMat);
    cone.position.y = 0.45;
    cone.rotation.x = Math.PI;
    group.add(cone);

    const beamGeo = new THREE.CylinderGeometry(0.015, 0.015, 0.9, 8);
    const beamMat = new THREE.MeshBasicMaterial({
      color: conf.color,
      transparent: true,
      opacity: 0.5,
    });
    const beam = new THREE.Mesh(beamGeo, beamMat);
    beam.position.y = 0.0;
    group.add(beam);

    const ringGeo = new THREE.TorusGeometry(0.28, 0.02, 8, 32);
    const ringMat = new THREE.MeshBasicMaterial({
      color: conf.color,
      transparent: true,
      opacity: 0.8,
    });
    const ring = new THREE.Mesh(ringGeo, ringMat);
    ring.rotation.x = Math.PI / 2;
    ring.position.y = -0.42;
    group.add(ring);
    group.userData.ring = ring;

    group.position.copy(point);
    return group;
  }, []);

  const addMarker = useCallback(
    (point, mode, range) => {
      if (!markersGroupRef.current) return;
      const mesh = buildMarkerMesh(point, mode);
      markersGroupRef.current.add(mesh);
      const id = `${Date.now()}-${Math.random().toString(36).slice(2, 7)}`;
      mesh.userData.id = id;
      setMarkers((prev) => [
        ...prev,
        { id, mode, range: range.toFixed(1) },
      ]);
    },
    [buildMarkerMesh]
  );

  const handlePlacement = useCallback(
    (evt) => {
      if (!cameraRef.current) return;
      const { locationX, locationY } = evt.nativeEvent;
      const ndcX = (locationX / SCREEN_W) * 2 - 1;
      const ndcY = -(locationY / SCREEN_H) * 2 + 1;

      const raycaster = new THREE.Raycaster();
      raycaster.setFromCamera(new THREE.Vector2(ndcX, ndcY), cameraRef.current);

      const groundPlane = new THREE.Plane(
        new THREE.Vector3(0, 1, 0),
        -GROUND_Y
      );
      const point = new THREE.Vector3();
      const hit = raycaster.ray.intersectPlane(groundPlane, point);
      if (!hit) return;

      const distance = cameraRef.current.position.distanceTo(point);
      const rangeMeters = Math.max(0.5, distance * 2.2);

      addMarker(point, targetMode, rangeMeters);
      setLastRange(rangeMeters);
      setTargetStatus('LOCKED');
      if (lockTimeout.current) clearTimeout(lockTimeout.current);
      lockTimeout.current = setTimeout(() => setTargetStatus('SCANNING'), 1800);
    },
    [targetMode, addMarker]
  );

  const clearMarkers = useCallback(() => {
    if (markersGroupRef.current) {
      [...markersGroupRef.current.children].forEach((child) => {
        markersGroupRef.current.remove(child);
        child.traverse((n) => {
          if (n.geometry) n.geometry.dispose();
          if (n.material) n.material.dispose();
        });
      });
    }
    setMarkers([]);
    setLastRange(null);
    setTargetStatus('SCANNING');
  }, []);

  const onContextCreate = useCallback(async (gl) => {
    const { drawingBufferWidth: width, drawingBufferHeight: height } = gl;
    const renderer = new Renderer({ gl });
    renderer.setSize(width, height);
    renderer.setPixelRatio(1);
    renderer.setClearColor(0x000000, 0);

    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(65, width / height, 0.1, 1000);
    camera.position.set(0, 0.6, 0);
    camera.rotation.x = -0.12;

    const ambient = new THREE.AmbientLight(0xffffff, 1.1);
    scene.add(ambient);
    const directional = new THREE.DirectionalLight(0xffffff, 0.6);
    directional.position.set(2, 5, 2);
    scene.add(directional);

    const grid = new THREE.GridHelper(220, 60, 0x00eaff, 0x0a3d3c);
    grid.position.y = GROUND_Y;
    grid.material.transparent = true;
    grid.material.opacity = 0.35;
    scene.add(grid);

    const markersGroup = new THREE.Group();
    scene.add(markersGroup);

    sceneRef.current = scene;
    cameraRef.current = camera;
    rendererRef.current = renderer;
    glRef.current = gl;
    markersGroupRef.current = markersGroup;

    const render = () => {
      animFrame.current = requestAnimationFrame(render);
      const tilt = deviceTiltRef.current;
      camera.rotation.x = -0.12 + tilt.x * 0.15;
      camera.rotation.y = tilt.y * 0.2;

      markersGroup.children.forEach((m) => {
        m.rotation.y += 0.02;
        if (m.userData.ring) m.userData.ring.rotation.z += 0.03;
      });

      renderer.render(scene, camera);
      gl.endFrameEXP();
    };
    render();
  }, []);

  if (!permission) {
    return <View style={styles.fallback} />;
  }

  if (!permission.granted) {
    return (
      <View style={styles.fallback}>
        <Text style={styles.fallbackTitle}>STRIKE ZONE AR</Text>
        <Text style={styles.fallbackText}>
          Camera access is required to scan for strike zones.
        </Text>
        <TouchableOpacity style={styles.grantButton} onPress={requestPermission}>
          <Text style={styles.grantButtonText}>GRANT CAMERA ACCESS</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <StatusBar hidden />
      <CameraView style={StyleSheet.absoluteFill} facing="back" />
      <GLView style={StyleSheet.absoluteFill} onContextCreate={onContextCreate} />

      <Pressable style={StyleSheet.absoluteFill} onPress={handlePlacement} />

      <View style={styles.reticle} pointerEvents="none">
        <View style={styles.reticleRing} />
        <View style={styles.reticleLineH} />
        <View style={styles.reticleLineV} />
      </View>

      <View style={styles.overlay} pointerEvents="box-none">
        {hudVisible && (
          <View style={styles.topHud}>
            <View style={styles.hudCell}>
              <Text style={styles.hudLabel}>HEADING</Text>
              <Text style={styles.hudValue}>
                {compassLabel(heading)} {String(heading).padStart(3, '0')}°
              </Text>
            </View>
            <View style={styles.hudCell}>
              <Text style={styles.hudLabel}>STATUS</Text>
              <Text
                style={[
                  styles.hudValue,
                  {
                    color: targetStatus === 'LOCKED' ? '#39ff14' : '#ffee00',
                  },
                ]}
              >
                {targetStatus}
              </Text>
            </View>
            <View style={styles.hudCell}>
              <Text style={styles.hudLabel}>RANGE</Text>
              <Text style={styles.hudValue}>
                {lastRange ? `${lastRange.toFixed(1)}m` : '--'}
              </Text>
            </View>
          </View>
        )}

        <View style={styles.markerCounter}>
          <Text style={styles.markerCounterText}>
            MARKERS: {markers.length}
          </Text>
          <Text
            style={[styles.markerCounterText, { color: MODES[targetMode].hex }]}
          >
            MODE: {MODES[targetMode].label}
          </Text>
        </View>

        <View style={styles.bottomBar}>
          <View style={styles.modeRow}>
            {Object.keys(MODES).map((key) => {
              const active = key === targetMode;
              const conf = MODES[key];
              return (
                <TouchableOpacity
                  key={key}
                  style={[
                    styles.modeButton,
                    {
                      borderColor: conf.hex,
                      backgroundColor: active ? conf.hex + '33' : 'transparent',
                    },
                  ]}
                  onPress={() => setTargetMode(key)}
                >
                  <Text style={[styles.modeButtonText, { color: conf.hex }]}>
                    {conf.label}
                  </Text>
                </TouchableOpacity>
              );
            })}
          </View>
          <View style={styles.actionRow}>
            <TouchableOpacity style={styles.actionButton} onPress={clearMarkers}>
              <Text style={styles.actionButtonText}>CLEAR MARKERS</Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={styles.actionButton}
              onPress={() => setHudVisible((v) => !v)}
            >
              <Text style={styles.actionButtonText}>
                {hudVisible ? 'HIDE HUD' : 'SHOW HUD'}
              </Text>
            </TouchableOpacity>
          </View>
        </View>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#000',
  },
  fallback: {
    flex: 1,
    backgroundColor: '#000',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 32,
  },
  fallbackTitle: {
    color: '#00eaff',
    fontSize: 24,
    fontWeight: '800',
    letterSpacing: 2,
    marginBottom: 12,
    fontFamily: Platform.OS === 'ios' ? 'Courier' : 'monospace',
  },
  fallbackText: {
    color: '#aaa',
    fontSize: 14,
    textAlign: 'center',
    marginBottom: 24,
  },
  grantButton: {
    borderWidth: 1,
    borderColor: '#00eaff',
    paddingVertical: 12,
    paddingHorizontal: 24,
    borderRadius: 4,
  },
  grantButtonText: {
    color: '#00eaff',
    fontWeight: '700',
    letterSpacing: 1,
  },
  overlay: {
    ...StyleSheet.absoluteFillObject,
    justifyContent: 'space-between',
  },
  topHud: {
    marginTop: Platform.OS === 'ios' ? 50 : 30,
    marginHorizontal: 14,
    flexDirection: 'row',
    justifyContent: 'space-between',
    backgroundColor: 'rgba(0,10,12,0.55)',
    borderWidth: 1,
    borderColor: '#0f3d3d',
    borderRadius: 6,
    paddingVertical: 8,
    paddingHorizontal: 12,
  },
  hudCell: {
    alignItems: 'center',
    flex: 1,
  },
  hudLabel: {
    color: '#4fd8d8',
    fontSize: 10,
    letterSpacing: 1.5,
    fontFamily: Platform.OS === 'ios' ? 'Courier' : 'monospace',
  },
  hudValue: {
    color: '#00eaff',
    fontSize: 15,
    fontWeight: '700',
    marginTop: 2,
    fontFamily: Platform.OS === 'ios' ? 'Courier' : 'monospace',
  },
  markerCounter: {
    alignSelf: 'center',
    flexDirection: 'row',
    gap: 16,
    backgroundColor: 'rgba(0,10,12,0.5)',
    borderRadius: 4,
    paddingVertical: 4,
    paddingHorizontal: 12,
    marginBottom: 8,
  },
  markerCounterText: {
    color: '#00eaff',
    fontSize: 11,
    fontFamily: Platform.OS === 'ios' ? 'Courier' : 'monospace',
    letterSpacing: 1,
  },
  reticle: {
    position: 'absolute',
    top: SCREEN_H / 2 - 30,
    left: SCREEN_W / 2 - 30,
    width: 60,
    height: 60,
    alignItems: 'center',
    justifyContent: 'center',
  },
  reticleRing: {
    position: 'absolute',
    width: 44,
    height: 44,
    borderRadius: 22,
    borderWidth: 1.5,
    borderColor: 'rgba(0,234,255,0.6)',
  },
  reticleLineH: {
    position: 'absolute',
    width: 60,
    height: 1,
    backgroundColor: 'rgba(0,234,255,0.4)',
  },
  reticleLineV: {
    position: 'absolute',
    width: 1,
    height: 60,
    backgroundColor: 'rgba(0,234,255,0.4)',
  },
  bottomBar: {
    marginBottom: Platform.OS === 'ios' ? 34 : 20,
    marginHorizontal: 14,
    backgroundColor: 'rgba(0,10,12,0.6)',
    borderWidth: 1,
    borderColor: '#0f3d3d',
    borderRadius: 8,
    padding: 10,
  },
  modeRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 8,
    gap: 8,
  },
  modeButton: {
    flex: 1,
    borderWidth: 1,
    borderRadius: 6,
    paddingVertical: 10,
    alignItems: 'center',
  },
  modeButtonText: {
    fontSize: 11,
    fontWeight: '700',
    letterSpacing: 0.5,
  },
  actionRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    gap: 8,
  },
  actionButton: {
    flex: 1,
    backgroundColor: 'rgba(255,238,0,0.08)',
    borderWidth: 1,
    borderColor: '#ffee00',
    borderRadius: 6,
    paddingVertical: 10,
    alignItems: 'center',
  },
  actionButtonText: {
    color: '#ffee00',
    fontSize: 11,
    fontWeight: '700',
    letterSpacing: 0.5,
  },
});
