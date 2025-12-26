import React, { useMemo, useRef, useState, useEffect } from "react";
import { Canvas, useFrame } from "@react-three/fiber";
import {
  Environment,
  OrbitControls,
  EffectComposer,
  Bloom,
  ToneMapping,
} from "@react-three/postprocessing";
import * as THREE from "three";
import { random } from "lodash"; // 假定有 lodash 或手写随机函数

// --- ⚙️ 配置参数 (Configuration) ---
const CONFIG = {
  count: 3000, // 粒子数量，越多越奢华
  colors: {
    emerald: new THREE.Color("#004225").convertSRGBToLinear(), // 祖母绿
    gold: new THREE.Color("#FFD700").convertSRGBToLinear(), // 奢华金
  },
  physics: {
    stiffness: 0.08, // 弹簧硬度 (回归树形的速度)
    damping: 0.92, // 阻尼 (空气摩擦，越小停得越快)
    explosionForce: 2.5, // 炸裂时的初始冲击力
  },
};

type TreeState = "SCATTERED" | "TREE_SHAPE";

// --- 🎄 核心组件 (The Artifact) ---
const ArixLuxuryTree = () => {
  const meshRef = useRef < THREE.InstancedMesh > null;
  const [treeState, setTreeState] = useState < TreeState > "TREE_SHAPE";

  // --- 1. 数据初始化 & 双位置系统 (Dual Position System) ---
  const data = useMemo(() => {
    const tempObj = new THREE.Object3D();
    const count = CONFIG.count;

    // 物理状态存储 (CPU side physics buffers)
    const positions = new Float32Array(count * 3); // 当前位置
    const velocities = new Float32Array(count * 3); // 当前速度
    const targets = new Float32Array(count * 3); // 目标位置 (根据状态切换)

    // 旋转物理
    const quaternions = new Float32Array(count * 4); // 当前旋转
    const angularVelocities = new Float32Array(count * 3); // 角速度

    // 静态属性
    const treeCoords = new Float32Array(count * 3); // 预计算：树形态坐标
    const scatterCoords = new Float32Array(count * 3); // 预计算：散落形态坐标
    const colors = new Float32Array(count * 3);

    for (let i = 0; i < count; i++) {
      const i3 = i * 3;
      const i4 = i * 4;

      // A. 生成树形态 (圆锥螺旋分布)
      const theta = Math.random() * Math.PI * 2 * 10; // 螺旋圈数
      const y = Math.random() * 10 - 5; // 高度范围 [-5, 5]
      const radius = (5 - y) * 0.4 * Math.random(); // 底部宽，顶部窄
      treeCoords[i3] = Math.cos(theta) * radius;
      treeCoords[i3 + 1] = y;
      treeCoords[i3 + 2] = Math.sin(theta) * radius;

      // B. 生成散落形态 (球体随机分布)
      const r = 15 + Math.random() * 10; // 半径 15-25 的大球壳
      const phi = Math.acos(2 * Math.random() - 1);
      const thetaSph = Math.random() * Math.PI * 2;
      scatterCoords[i3] = r * Math.sin(phi) * Math.cos(thetaSph);
      scatterCoords[i3 + 1] = r * Math.sin(phi) * Math.sin(thetaSph);
      scatterCoords[i3 + 2] = r * Math.cos(phi);

      // C. 初始化物理状态
      // 初始位置设为树形态
      positions.set(
        [treeCoords[i3], treeCoords[i3 + 1], treeCoords[i3 + 2]],
        i3
      );

      // 随机旋转
      const q = new THREE.Quaternion().setFromEuler(
        new THREE.Euler(Math.random() * Math.PI, Math.random() * Math.PI, 0)
      );
      quaternions.set([q.x, q.y, q.z, q.w], i4);

      // D. 奢华配色 (20% 金，80% 绿)
      const isGold = Math.random() > 0.8;
      const color = isGold ? CONFIG.colors.gold : CONFIG.colors.emerald;
      colors[i3] = color.r;
      colors[i3 + 1] = color.g;
      colors[i3 + 2] = color.b;
    }

    return {
      positions,
      velocities,
      targets,
      treeCoords,
      scatterCoords,
      quaternions,
      angularVelocities,
      colors,
    };
  }, []);

  // --- 2. 状态切换触发器 (State Trigger) ---
  useEffect(() => {
    const count = CONFIG.count;
    const {
      velocities,
      angularVelocities,
      positions,
      treeCoords,
      scatterCoords,
    } = data;

    for (let i = 0; i < count; i++) {
      const i3 = i * 3;

      // 设置新的目标位置
      const dest = treeState === "TREE_SHAPE" ? treeCoords : scatterCoords;
      data.targets[i3] = dest[i3];
      data.targets[i3 + 1] = dest[i3 + 1];
      data.targets[i3 + 2] = dest[i3 + 2];

      // 🔥 物理炸裂逻辑 (Explosion Physics)
      // 如果是从 树 -> 散落，施加巨大的向外斥力
      if (treeState === "SCATTERED") {
        const x = positions[i3];
        const y = positions[i3 + 1];
        const z = positions[i3 + 2];

        // 计算从中心向外的法向量
        const len = Math.sqrt(x * x + y * y + z * z) || 1;
        const dirX = x / len;
        const dirY = y / len;
        const dirZ = z / len;

        // 施加瞬时速度 (Impulse)
        velocities[i3] +=
          dirX * CONFIG.physics.explosionForce * (0.5 + Math.random());
        velocities[i3 + 1] +=
          dirY * CONFIG.physics.explosionForce * (0.5 + Math.random());
        velocities[i3 + 2] +=
          dirZ * CONFIG.physics.explosionForce * (0.5 + Math.random());

        // 施加随机旋转力矩
        angularVelocities[i3] = (Math.random() - 0.5) * 0.5;
        angularVelocities[i3 + 1] = (Math.random() - 0.5) * 0.5;
        angularVelocities[i3 + 2] = (Math.random() - 0.5) * 0.5;
      }
    }
  }, [treeState, data]);

  // --- 3. 物理帧循环 (The Physics Loop) ---
  useFrame(() => {
    if (!meshRef.current) return;

    const { positions, velocities, targets, quaternions, angularVelocities } =
      data;

    const count = CONFIG.count;
    const dummy = new THREE.Object3D();
    const tempQ = new THREE.Quaternion();

    // 弹簧参数
    const k = CONFIG.physics.stiffness;
    const d = CONFIG.physics.damping;

    for (let i = 0; i < count; i++) {
      const i3 = i * 3;
      const i4 = i * 4;

      // --- 位置物理 (Spring Force) ---
      // Force = (Target - Current) * k
      const ax = (targets[i3] - positions[i3]) * k;
      const ay = (targets[i3 + 1] - positions[i3 + 1]) * k;
      const az = (targets[i3 + 2] - positions[i3 + 2]) * k;

      // Velocity += Force
      velocities[i3] += ax;
      velocities[i3 + 1] += ay;
      velocities[i3 + 2] += az;

      // Velocity *= Damping (摩擦力)
      velocities[i3] *= d;
      velocities[i3 + 1] *= d;
      velocities[i3 + 2] *= d;

      // Position += Velocity
      positions[i3] += velocities[i3];
      positions[i3 + 1] += velocities[i3 + 1];
      positions[i3 + 2] += velocities[i3 + 2];

      // --- 旋转物理 (Angular Physics) ---
      // 简单模拟：速度越大，旋转越快；随着速度减慢，旋转也减慢
      angularVelocities[i3] *= 0.95; // 角阻尼
      angularVelocities[i3 + 1] *= 0.95;
      angularVelocities[i3 + 2] *= 0.95;

      // 应用旋转
      tempQ.setFromEuler(
        new THREE.Euler(
          angularVelocities[i3],
          angularVelocities[i3 + 1],
          angularVelocities[i3 + 2]
        )
      );

      const currentQ = new THREE.Quaternion(
        quaternions[i4],
        quaternions[i4 + 1],
        quaternions[i4 + 2],
        quaternions[i4 + 3]
      );
      currentQ.multiply(tempQ); // 累加旋转
      currentQ.normalize();

      quaternions[i4] = currentQ.x;
      quaternions[i4 + 1] = currentQ.y;
      quaternions[i4 + 2] = currentQ.z;
      quaternions[i4 + 3] = currentQ.w;

      // --- 更新矩阵 ---
      dummy.position.set(positions[i3], positions[i3 + 1], positions[i3 + 2]);
      dummy.quaternion.copy(currentQ);

      // 树形态时稍微大一点，散落时保持原大
      const scale = treeState === "TREE_SHAPE" ? 1 : 0.8;
      dummy.scale.setScalar(scale);

      dummy.updateMatrix();
      meshRef.current.setMatrixAt(i, dummy.matrix);
    }

    meshRef.current.instanceMatrix.needsUpdate = true;
  });

  return (
    <>
      <instancedMesh
        ref={meshRef}
        args={[undefined, undefined, CONFIG.count]}
        onClick={() =>
          setTreeState((s) => (s === "TREE_SHAPE" ? "SCATTERED" : "TREE_SHAPE"))
        }
        // 增加一点 hover 效果或者 cursor change 会更好
        onPointerOver={() => (document.body.style.cursor = "pointer")}
        onPointerOut={() => (document.body.style.cursor = "auto")}
      >
        {/* 使用棱锥体 (Tetrahedron) 或 细长Box 模拟宝石/金针碎片 */}
        <cylinderGeometry args={[0.02, 0.1, 0.8, 4]} />
        <meshStandardMaterial
          roughness={0.15}
          metalness={0.9}
          envMapIntensity={1.5}
        />
        <instancedBufferAttribute
          attach="geometry-attributes-color"
          args={[data.colors, 3]}
        />
      </instancedMesh>
    </>
  );
};

// --- 🎬 主场景 (Main Scene) ---
export default function Scene() {
  return (
    <div style={{ width: "100vw", height: "100vh", background: "#050505" }}>
      <Canvas
        camera={{ position: [0, 0, 15], fov: 45 }}
        gl={{ antialias: false }}
      >
        {/* 环境光照 */}
        <Environment preset="city" />
        <ambientLight intensity={0.2} color="#004225" />
        <pointLight position={[10, 10, 10]} intensity={1} color="#FFD700" />

        <ArixLuxuryTree />

        <OrbitControls enablePan={false} autoRotate autoRotateSpeed={0.5} />

        {/* 电影感后期 */}
        <EffectComposer disableNormalPass>
          <Bloom
            luminanceThreshold={0.8} // 只有高亮部分发光（金色）
            mipmapBlur
            intensity={1.2}
            radius={0.6}
          />
          <ToneMapping />
        </EffectComposer>
      </Canvas>

      {/* UI Overlay */}
      <div
        style={{
          position: "absolute",
          bottom: 40,
          left: "50%",
          transform: "translateX(-50%)",
          color: "#FFD700",
          fontFamily: "serif",
          letterSpacing: "2px",
          pointerEvents: "none",
        }}
      >
        ARIX SIGNATURE • TAP TO INTERACT
      </div>
    </div>
  );
}
