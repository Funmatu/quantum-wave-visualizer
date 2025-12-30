# 6D Quantum Hyper-Spatial Visualizer

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Technology: Three.js](https://img.shields.io/badge/Tech-Three.js_r128-black)
![Shader: GLSL](https://img.shields.io/badge/Shader-GLSL_ES_3.0-purple)
![Version: 2.1](https://img.shields.io/badge/Version-2.1-green)

<div align="center">
  <h3>
    <a href="https://funmatu.github.io/quantum-wave-visualizer/">🚀 Launch Live Demonstration</a>
  </h3>
  <p>
    Real-time GPU simulation of Schrödinger wave packet evolution in 6-dimensional phase space.
  </p>
</div>

---

## 📖 概要 (Overview)

**6D Quantum Hyper-Spatial Visualizer** は、Webブラウザ上で動作する高度な物理シミュレーター兼ビジュアライザーです。
量子力学における自由粒子の時間発展（Time Evolution of a Free Particle）を記述するシュレーディンガー方程式を、GPU（GLSL）を用いてリアルタイムに計算し、その振る舞いを視覚化します。

本プロジェクトは、量子力学が持つ**6次元の情報**（空間3次元 + 複素数2次元 + 時間1次元）を、人間の知覚可能な3次元グラフィックスへと射影（Projection）するための実験的なインターフェースを提供します。

### 核心となる機能
*   **物理空間 (R³)** における確率密度 $|\Psi|^2$ のボリューメトリックな可視化。
*   **位相空間 (Phase Space)** における複素波動関数（実部・虚部）の螺旋構造の可視化。
*   **不確定性原理** に基づく波束の分散（拡散）過程のシミュレーション。
*   **ポストプロセス** による光学的ブルーム処理を用いた「発光する量子」の表現。

---

## 🌌 物理学的背景 (Theoretical Background)

本シミュレーターは、以下の数理モデルに基づいて実装されています。

### 1. ガウス波束の定義
初期状態（ $t=0$ ）における波動関数 $\Psi(\vec{r}, 0)$ は、ガウス分布に従う波束として定義されます。

$$
\Psi(\vec{r}, 0) = \frac{1}{(\pi \sigma_0^2)^{3/4}} \exp\left( -\frac{|\vec{r}|^2}{2\sigma_0^2} + i\vec{k}_0 \cdot \vec{r} \right)
$$

ここで、 $\sigma_0$ は初期の波束の幅、 $\vec{k}_0$ は初期運動量ベクトルを表します。

### 2. 時間発展と分散 (Time Evolution & Dispersion)
自由粒子のハミルトニアン $\hat{H} = -\frac{\hbar^2}{2m}\nabla^2$ に従い、時間 $t$ における波動関数は分散を起こしながら広がります。本プログラムでは、以下の解析解をシェーダー内で計算しています。

時間依存の複素分散項 $\Sigma(t)$：  

$$
\Sigma(t)^2 = \sigma_0^2 + i \frac{\hbar t}{m}
$$

(※コード内では `uDispersion` パラメータとして係数を簡略化)  

これにより、確率密度関数 $|\Psi|^2$ は時間が経過するにつれて空間的に広がり、中心付近の振幅が減少します。これはハイゼンベルクの不確定性原理 $\Delta x \Delta p \ge \hbar/2$ の視覚的表現でもあります。

### 3. 次元のマッピング (Dimensional Mapping)
本システムにおける「6D」の定義は以下の通りです。

| 次元 | 変数 | 物理的意味 | ビジュアライゼーションへの適用 |
|:---:|:---:|:---|:---|
| **1-3** | $x, y, z$ | 物理的空間座標 | パーティクルのワールド座標 (Geometry) |
| **4** | $Re(\Psi)$ | 波動関数の実部 | 複素空間モード時のY軸変位 / 色相計算 |
| **5** | $Im(\Psi)$ | 波動関数の虚部 | 複素空間モード時のZ軸変位 / 色相計算 |
| **6** | $t$ | 時間 | `uTime` Uniformによるアニメーション進行 |

---

## 🛠 技術アーキテクチャ (Technical Architecture)

### システム構成
*   **Core Framework**: [Three.js (r128)](https://threejs.org/)
*   **Language**: JavaScript (ES6+), GLSL (OpenGL Shading Language)
*   **UI/Control**: dat.GUI, Tailwind CSS
*   **Animation**: GSAP (Loading transitions), requestAnimationFrame

### レンダリングパイプライン詳細

#### 1. ジオメトリ生成 (Spherical Uniform Distribution)
従来の立方体ランダム分布で発生する「キューブ状のアーティファクト」を排除するため、本システムでは**球体内部への一様乱数生成アルゴリズム**を採用しています。
`Math.cbrt()`（立方根）を用いることで、中心への過度な集中を防ぎ、体積に対して均等な密度を持つ粒子群（120,000点）を生成します。

```javascript
// アルゴリズム抜粋
const r = Math.cbrt(Math.random()) * radiusRange; // 体積一様分布
const theta = Math.random() * Math.PI * 2;
const phi = Math.acos((Math.random() * 2) - 1);
```

#### 2. Vertex Shader (物理演算層)
CPU側ではなく、GPUの並列処理能力を利用して各パーティクルの位置と状態を計算します。
*   **モード0 (Physical Cloud)**:
    *   座標：元の空間座標 $(x,y,z)$ に、位相に応じた微細な振動を加算。
    *   計算：ガウス関数の振幅計算により、中心から離れた粒子をフィルタリング。
*   **モード1 (Complex Plane Projection)**:
    *   座標：X軸は空間座標を維持。Y軸に実部 $Re(\Psi)$、Z軸に虚部 $Im(\Psi)$ をマッピング。
    *   結果：波動関数がコルクスクリュー状に回転しながら進む「オイラーの公式 $e^{ix} = \cos x + i\sin x$」の幾何学的表現を描画。

#### 3. Fragment Shader (スペクトルレンダリング)
*   **位相のカラーマッピング**: 波動関数の位相角（Phase Angle）を計算し、HSV色空間の Hue（色相）に変換。$0 \to 2\pi$ の変化が全スペクトル色に対応します。
*   **ソフトパーティクル**: 各点は中心が濃く、縁が薄い円形として描画され、重なり合うことで滑らかなガス状の質感を生成します。

#### 4. Post-Processing (ブルーム効果)
`UnrealBloomPass` を使用し、閾値（Threshold）を超えた輝度を持つピクセルを周囲に拡散させます。
*   モード切替時にブルーム強度を動的に調整することで、物理空間モードでは「輝くエネルギー体」、複素空間モードでは「精緻な数理モデル」として最適な視認性を提供します。

---

## 🎮 操作方法 (Controls)

画面右上のコントロールパネル（dat.GUI）およびマウス操作により、シミュレーションに介入できます。

### マウス操作
*   **左ドラッグ**: カメラの回転 (Orbit)
*   **右ドラッグ**: カメラの位置移動 (Pan)
*   **スクロール**: ズームイン/アウト

### パラメータ設定
#### Wave Packet Physics
*   **Uncertainty $\sigma$**: 波束の初期幅。値が小さいほど位置が特定され（鋭くなり）、運動量の不確定性が増します。
*   **Momentum $k_x$**: 波束の進行方向と速度。
*   **Dispersion Factor**: 時間経過に伴う拡散の速さ。
*   **Reset Evolution**: 時間 $t=0$ にリセットします。

#### Dimensional Projection
*   **View Mode**:
    *   `Physical Cloud (XYZ)`: 我々が観測する確率密度の雲。
    *   `Complex Spiral (X/Re/Im)`: 量子力学の計算空間（位相空間）の可視化。
*   **Bloom Intensity**: 発光エフェクトの強さ調整。

---

## 💻 開発とインストール (Development)

本プロジェクトは静的なWebサイトとして構成されているため、特別なビルドプロセスは不要です。

### ローカルでの実行方法

1.  **リポジトリのクローン**
    ```bash
    git clone https://github.com/FUNMATU/quantum-wave-visualizer.git
    cd quantum-wave-visualizer
    ```

2.  **ローカルサーバーの起動**
    セキュリティ制約（CORS等）を回避し、テクスチャやシェーダーを正しく読み込むため、ローカルサーバー経由で開くことを推奨します。

    *   VS Codeを使用している場合: "Live Server" 拡張機能を使用。
    *   Pythonがインストールされている場合:
        ```bash
        # Python 3.x
        python -m http.server 8000
        ```

3.  **ブラウザでアクセス**
    `http://localhost:8000` を開いてください。

---

## 🔮 今後の展望 (Roadmap)

*   **ポテンシャル障壁の導入**: トンネル効果のシミュレーション。
*   **二重スリット実験**: 干渉縞のリアルタイム生成。
*   **WebXR対応**: VR/ARデバイスでの没入型量子体験。
*   **GPU Compute Shader化**: 100万パーティクル以上へのスケールアップ。

---

## 📜 ライセンス (License)

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

Copyright (c) 2025 FUNMATU
