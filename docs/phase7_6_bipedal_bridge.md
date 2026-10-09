# フェーズ7-6 手順書: 二足歩行への橋渡し

[`docs/learning_plan.md`](learning_plan.md) フェーズ7の7-6（[`docs/idea_origin.md`](idea_origin.md) ステップ7）に対応する。フェーズ7-0〜7-5では、車両の速度の追従を題材に、強化学習でゲインを選び、NNの方策を学習させ、ROS2のノードとして動かした。このページでは、その先にある二足歩行のロボットに目を向ける。MuJoCoの2次元の二足（`Walker2d`）を、これまでと同じSB3のSACで短く学習させて、車両との違いを体験する。そのうえで、二足歩行に進むために足りない知識と、使われている道具、要るPCの性能を整理する。

- 想定環境: WSL2 + Ubuntu 24.04（ROS2は使わない。強化学習の仮想環境だけを使う）
- 前提: フェーズ7-0（[`docs/phase7_0_rl_intro.md`](phase7_0_rl_intro.md)。`~/rl_venv` にSB3とGymnasiumがあり、SACの使い方が分かる）。7-1〜7-5を読んでいると、5節の比べ方が分かりやすい
- 所要目安: 2コマ（計算を待つ時間が約20分ある）
- 言語: Python（ROS2のノードは書かない）

> **この手順書の位置づけ（暫定）**: フェーズ7は作成中で、構成を大きく改める可能性がある。このページは、二足歩行に本格的に取り組む前の橋渡しで、後半（6節）は読み物である。

> **進め方**: 1節で全体像を見る。2節で、仮想環境にMuJoCoを入れる。3節で、二足の環境を確かめ、4節でSACに学習させる。5節で、車両の題材との違いを整理し、6節で、二足歩行に進むための知識・道具・PCの性能を読む。サンプルは学習の手がかりとして最小限に書いたもので、公式の文書の転載ではない。

> **実行環境が無くても読めるように**: 実行する手順の直後には「期待する結果」として、表示される内容の例とその読み方を載せている。期待する結果は、筆者の環境（2026-10-08。SB3 2.9.0・Gymnasium 1.3.0・MuJoCo 3.15.0・PyTorch 2.14.1（CPU版））で実際に実行した表示である。作成時は、`~/rl_venv` を写した使い捨ての仮想環境にMuJoCoを入れて確かめた。3節の表示は、同じ版なら何度実行しても同じになる（2回実行して確かめた）。4節の学習は、同じPC・同じ版・同じ設定なら同じ値になる見込みだが、PCや版、計算に使うスレッドの数が違うと変わる（7-4の6節）。`time[s]` の列は、実行ごとに変わる。4-4節の任意の「動きを見る」は、ウィンドウを開くため、作成時には実行していない（ウィンドウを開かずに、同じスクリプトが最後まで走ることは確かめた）。

## 0. 学習目標と完了条件

1. 車両の速度の追従と、二足の歩行とで、強化学習の問題の形（観測と行動の数、エピソードの終わり方、報酬の作り方、止まっていても安定か）がどう違うかを説明できる。
2. MuJoCoの二足の環境を、これまでと同じSB3のSACで学習させ、学習の量と、立っていられる長さ・進む距離の関係を読み取れる。
3. 二足歩行に進むために足りない知識と、使われている道具（シミュレータ・強化学習のライブラリ・ROS2の部品）、要るPCの性能を説明できる。

完了条件: 4節で、Walker2dを10万ステップ学習させ（または期待する結果を読み）、まだ歩けていない理由を、収益の内訳と学習の量から説明できる。6節を読み、次に何を学べばよいかを、自分の言葉で書ける。

## 1. 全体像

このプロジェクトのアイデア（[`docs/idea_origin.md`](idea_origin.md)）は、最初から二足歩行のロボットを見据えていた。ただ、二足歩行は、計算の負荷も、要る知識も大きい。そこで、まず慣性のある車両の速度の追従を題材にして、ROS2・物理シミュレータ・記録と評価・強化学習を一通り学んできた。

二足歩行では、何が変わるのか。いちばん大きいのは、**何もしないと倒れる**ことである。車両は、ペダルを離せば、やがて止まって落ち着く。二足のロボットは、関節に力を入れ続けないと、すぐに倒れる。しかも、関節は1つではなく、全身の関節の力を、同時に、細かく合わせる必要がある。人がPI制御のような簡単な式を書くのが難しいので、強化学習がよく使われる分野の1つである（7-0の6-2節の「強化学習が力を発揮するのは、式を立てにくい問題（接触や摩擦が複雑、観測が多い）」）。

このページでは、まず、二足を小さく試す。GymnasiumにあるMuJoCo（物理シミュレータの1つ）の環境 `Walker2d` は、2次元（前後と上下）だけで動く、脚が2本の簡単なロボットである。これを、7-0〜7-4と同じSB3のSACで学習させ、車両とどれだけ勝手が違うかを体験する。そのうえで、本格的に二足歩行を学習させるときに使われる道具と、要るPCの性能を読む。

![車両の題材（7-3〜7-5。観測5・行動1、PI制御とNNを比べ、Gazebo・ROS2のノードで動かした）から、このページで二足を小さく試し（Walker2d、MuJoCo、観測17・行動6、作成に使ったPCのCPUで）、性能の高いPCのGPUで数千の環境を並べて大量に学習し（Isaac Lab・MuJoCo Playground、rsl_rl・skrl）、ROS2でロボットへつなぐ（URDF・tf2・ros2_control、Gazebo・実機）、という流れ](img/phase7_6_bridge.svg)

<details>
<summary>同じ図（mermaid版）</summary>

```mermaid
flowchart LR
    B1["車両（7-3〜7-5）<br/>観測5・行動1<br/>PI制御とNNを比べる<br/>Gazebo・ROS2のノード"]
    B2["二足を試す（7-6）<br/>Walker2d（MuJoCo）<br/>観測17・行動6<br/>作成に使ったPCのCPUで"]
    B3["大量に学習<br/>Isaac Lab・MuJoCo Playground<br/>rsl_rl・skrl<br/>数千の環境を並べる（GPU）"]
    B4["ROS2でロボットへ<br/>URDF・tf2<br/>ros2_control<br/>Gazebo・実機"]
    B1 --> B2 --> B3 --> B4
    classDef reading stroke-dasharray: 5 5
    class B3,B4 reading
```

（点線の2つ（大量に学習・ROS2でロボットへ）は、このページでは読み物として扱う。）

</details>

## 2. MuJoCoを入れる

MuJoCoは、Google DeepMindが公開している物理シミュレータで、ロボットの学習の研究でよく使われる（Apache License 2.0）。CPUで動き、Pythonから `pip` で入れられる。Gymnasiumの二足の環境を使うには、Gymnasiumの追加の部品（`[mujoco]`）として、MuJoCoのPythonのパッケージと、関係するパッケージを入れる。

強化学習の仮想環境（[強化学習の環境構築](setup_rl_sb3.md)の `~/rl_venv`）を有効にしたターミナルで、次を実行する。Gymnasiumの版は、環境構築の4節で入れた1.3.0にそろえる。

```bash
source ~/rl_venv/bin/activate

pip install "gymnasium[mujoco]==1.3.0"
```

**期待する結果**（抜粋。末尾の行だけを載せる）:

```text
Successfully installed absl-py-2.5.0 etils-1.14.0 glfw-2.10.2 imageio-2.38.0 mujoco-3.15.0 packaging-26.3 pillow-12.3.0 pyopengl-3.1.10 websockets-17.2 zipp-4.1.1
```

`mujoco-3.15.0` があれば成功である。Gymnasium本体やnumpyは、すでに入っているので、`Requirement already satisfied` と表示され、入れ直されない。並んだ版は作成時のもので、`mujoco` などは、入れた時期によって新しい版になることがある（Gymnasium 1.3.0の `[mujoco]` は、`mujoco` の2.1.5以上を求める）。`glfw`・`pyopengl` は、4-4節の任意の「動きを見る」で、ウィンドウに描くための部品である。

> **補足: 仮想環境に足すこと**
>
> - **一般的なこと**: 仮想環境に後からパッケージを足すと、すでに入っているパッケージの版が、足したパッケージの求めに合わせて変わることがある。
> - **このページの扱い**: 上の表示のとおり、作成時は、SB3・PyTorch・numpyの版は変わらなかった。`pip list` で、`stable-baselines3` が2.9.0のままかを確かめておくとよい。
> - **実務の目安**: 題材ごとに仮想環境を分けると、互いの版の影響を受けない。このページでは、1つの仮想環境で続けられるように、同じ `~/rl_venv` に足した。

## 3. 二足の環境を確かめる

### 3-1. 確かめるスクリプト（`check_walker.py`）

ファイル: `~/rl_practice/check_walker.py`（ファイルの置き方は [サンプルコードを練習環境に置く方法](howto_place_code.md)）

```python
# MuJoCoの2次元の二足（Walker2d-v5）の観測と行動の形を表示し、でたらめな行動で5回走らせて、倒れるまでの様子を見る。
import gymnasium as gym

env = gym.make('Walker2d-v5')
print('observation:', env.observation_space)
print('action:', env.action_space)
print('dt:', env.unwrapped.dt)
print('seed  return  length  time[s]')
for seed in range(5):
    obs, info = env.reset(seed=seed)
    env.action_space.seed(seed)
    total, n, done = 0.0, 0, False
    while not done:
        obs, reward, terminated, truncated, info = env.step(env.action_space.sample())
        total += reward
        n += 1
        done = terminated or truncated
    print(f'{seed:4d}  {total:6.2f}  {n:6d}  {n * env.unwrapped.dt:7.3f}')
```

**`check_walker.py` の解説**

- **`gym.make('Walker2d-v5')`**: Gymnasiumに登録されている、MuJoCoの2次元の二足の環境を作る（`v5` は版）。7-0の3節の `Pendulum-v1` と同じ使い方である。
- **観測と行動の形**: 7-0の3-2節と同じく、`observation_space`・`action_space` を表示する。`env.unwrapped.dt` は、1ステップがシミュレーションの何秒分かである。
- **でたらめな行動**: `env.action_space.sample()` で、行動の範囲からでたらめに選ぶ（7-0の3-1節と同じ）。`env.action_space.seed(seed)` で、でたらめの選び方も種で決め、何度実行しても同じ結果になるようにする。倒れるとエピソードが終わる（`terminated`）ので、終わるまでの長さと、そのときの時間を表示する。

### 3-2. 動かす

```bash
cd ~/rl_practice

python check_walker.py
```

**期待する結果**:

```text
observation: Box(-inf, inf, (17,), float64)
action: Box(-1.0, 1.0, (6,), float32)
dt: 0.008
seed  return  length  time[s]
   0   23.50      46    0.368
   1    0.61      17    0.136
   2    3.35      14    0.112
   3    0.84      28    0.224
   4    5.24      22    0.176
```

- **観測は17個**: Gymnasiumの公式の文書（10節）によると、胴体の高さと角度、両脚の6つの関節の角度（ここまでで位置が8つ）と、胴体の速度・角速度と6つの関節の角速度（速度が9つ）である。前後の位置そのものは、観測に入らない。
- **行動は6個**: 両脚の、ももの付け根・ひざ・足首の関節にかけるトルク（−1〜1）。車両のペダルは1つだった。
- **1ステップは0.008秒**: MuJoCoの物理を0.002秒ずつ4回進める（車両の環境の0.1秒ごとより、ずっと細かい）。1エピソードは最長1000ステップ（8秒）で、それより前に倒れると終わる。
- **でたらめな行動では、0.1〜0.4秒で倒れる**: 14〜46ステップで終わり、収益もほぼ0である。車両なら、ペダルをでたらめに踏んでも、90秒走り続けた（エピソードは時間で終わった）。二足では、倒れることが、そのままエピソードの終わりになる。

報酬は、Gymnasiumの公式の文書によると、1ステップごとに次の3つを足したものである。

- **立っている分**: 倒れていなければ1。胴体の高さが0.8〜2.0 m、角度が−1〜1ラジアンの範囲にあることが「倒れていない」条件で、範囲を外れるとエピソードが終わる。
- **前に進んだ分**: そのステップの前向きの速さ（進んだ距離 ÷ 0.008秒）。
- **力の罰**: トルクの大きさの二乗の和に0.001を掛けたものを引く。

車両の報酬（7-1の2-5節）は、目標との差と行き過ぎ、つまり目標への追従のよさだけを見ていた。二足の報酬は、「倒れない」「前に進む」「力を使いすぎない」の3つを同時に求める。

## 4. SACで学習させる

### 4-1. 学習のスクリプト（`train_walker.py`）

7-2の `train_gain.py` と同じ形で、学習用とは別の評価用の環境を作り、途中経過を表示する（Walker2dは、7-4のGazeboの環境と違い、同じプロセスの中に何個でも作れる）。

ファイル: `~/rl_practice/train_walker.py`

```python
# MuJoCoの2次元の二足（Walker2d-v5）を、SB3のSACで学習させ、途中経過を表示して保存する。
import argparse
import time

import gymnasium as gym
import numpy as np
from stable_baselines3 import SAC

EVAL_SEEDS = range(5)       # 評価に使う初期状態の種
REPORT_STEPS = 20000        # 途中経過を表示する間隔 [ステップ]


# 方策 model を、評価用の環境で種ごとに1エピソードずつ動かし、収益・長さ・進んだ距離の平均を返す。
def evaluate(model, env):
    returns, lengths, distances = [], [], []
    for seed in EVAL_SEEDS:
        obs, info = env.reset(seed=seed)
        start_x = env.unwrapped.data.qpos[0]
        total, n, done = 0.0, 0, False
        while not done:
            action, _ = model.predict(obs, deterministic=True)
            obs, reward, terminated, truncated, info = env.step(action)
            total += reward
            n += 1
            done = terminated or truncated
        returns.append(total)
        lengths.append(n)
        distances.append(env.unwrapped.data.qpos[0] - start_x)
    return np.mean(returns), np.mean(lengths), np.mean(distances)


# 引数のステップ数だけ学習させ、2万ステップごとに、評価の収益・長さ・距離・エントロピーの係数を表示する。
def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--steps', type=int, default=100000)
    parser.add_argument('--seed', type=int, default=0)
    args = parser.parse_args()

    env, eval_env = gym.make('Walker2d-v5'), gym.make('Walker2d-v5')
    model = SAC('MlpPolicy', env, verbose=0, seed=args.seed)

    start = time.monotonic()
    print(' steps   return  length  distance[m]  ent_coef  time[s]')
    for k in range(1, args.steps // REPORT_STEPS + 1):
        model.learn(total_timesteps=REPORT_STEPS, reset_num_timesteps=False)
        mean_return, length, distance = evaluate(model, eval_env)
        print(f'{k * REPORT_STEPS:6d}  {mean_return:7.1f}  {length:6.0f}  {distance:11.2f}'
              f'  {model.log_ent_coef.detach().exp().item():8.4f}  {time.monotonic() - start:7.0f}', flush=True)
    model.save(f'walker_sac_{args.steps}')


if __name__ == '__main__':
    main()
```

**`train_walker.py` の解説**

- **`evaluate`**: 評価用の環境で、5つの種（初期状態の小さなゆらぎが変わる）を1エピソードずつ走らせ、収益・長さ（倒れるまでのステップ数。最長1000）・進んだ距離の平均を返す。`deterministic=True` で、探索のばらつきを入れない（7-0の4-1節）。進んだ距離は、MuJoCoの内部の状態 `env.unwrapped.data.qpos[0]`（胴体の前後の位置）の、始めと終わりの差である。観測には前後の位置が入らないので、ここで直接読む。
- **学習と評価の繰り返し**: 7-2と同じく、`model.learn(..., reset_num_timesteps=False)` で2万ステップずつ学習させ、そのたびに評価する。
- **`SAC('MlpPolicy', env, verbose=0, seed=args.seed)`**: 7-0の4-2節・7-4の3-1節と同じ、既定の設定のSACである。観測が17個、行動が6個になっても、書き方は変わらない。NNの入口と出口の数は、SB3が環境の形から決める。
- **保存**: 最後に、`walker_sac_100000.zip` のような名前で保存する（4-4節の任意の「動きを見る」で使う）。

### 4-2. 10万ステップ学習させる

約20分かかる。

```bash
python train_walker.py
```

**期待する結果**（`time[s]` の列は、実行ごとに変わる）:

```text
 steps   return  length  distance[m]  ent_coef  time[s]
 20000    563.1     300         2.12    0.0437      250
 40000    625.4     280         2.77    0.0567      495
 60000   1010.8     508         4.05    0.0461      726
 80000    415.3     187         1.84    0.0473      950
100000    812.2     334         3.84    0.0460     1177
```

各行は、2万ステップごとの評価である。`return` は収益、`length` は倒れるまでのステップ数（最長1000）、`distance[m]` は進んだ距離、`ent_coef` はエントロピーの係数（7-2の3-1節）、`time[s]` は学習を始めてからの秒数で、どれも5つの種の平均である。

- **倒れずにいられる時間は延びる**: でたらめな行動では、14〜46ステップ（0.1〜0.4秒）で倒れた。2万ステップの学習で、300ステップ（2.4秒）まで延びる。
- **それでも、最後まで立っていられない**: 10万ステップでも、長さは334ステップ（約2.7秒）で、1000ステップ（8秒）には遠い。6万ステップで508ステップまで延びた後、8万ステップで187ステップに落ちるなど、学習は一直線には進まない（7-4の3-2節と同じ）。
- **収益の内訳**: 10万ステップの収益812は、立っていた分（334）と、前に進んだ分（3.84 m ÷ 0.008秒 ≈ 480）の和に、ほぼ等しい（力の罰は小さい）。約2.7秒で3.84 m、秒速約1.4 mで前に進んで倒れている。倒れるまでの短い間に、前へ倒れ込むように距離を稼いでいる、と読める。歩いている、とは言いにくい。

二足の環境では、10万ステップは、学習の入口にすぎない。作成時に、同じ設定で30万ステップまで学習させた結果を、4-3節の補足に載せる。

### 4-3. 補足: 学習を延ばすと

作成時に、同じスクリプトで `--steps 300000` を指定し、30万ステップまで学習させた（約58分）。10万ステップまでの行は、4-2節とまったく同じ値になった（同じPC・同じ版・同じ設定のため）。

```bash
python train_walker.py --steps 300000
```

**期待する結果**（`time[s]` の列は、実行ごとに変わる）:

```text
 steps   return  length  distance[m]  ent_coef  time[s]
 20000    563.1     300         2.12    0.0437      235
 40000    625.4     280         2.77    0.0567      465
 60000   1010.8     508         4.05    0.0461      695
 80000    415.3     187         1.84    0.0473      930
100000    812.2     334         3.84    0.0460     1163
120000    727.4     312         3.34    0.0486     1393
140000    938.5     361         4.64    0.0465     1619
160000    908.7     334         4.61    0.0484     1847
180000   1180.7     373         6.48    0.0486     2079
200000   1041.8     360         5.47    0.0512     2310
220000   1105.5     444         5.31    0.0500     2539
240000   1211.1     403         6.49    0.0514     2769
260000   1198.1     447         6.03    0.0513     2998
280000   1329.8     419         7.31    0.0518     3231
300000   1228.5     461         6.16    0.0502     3465
```

- **少しずつ伸びるが、まだ歩けていない**: 30万ステップの収益は1229で、10万ステップ（812）の約1.5倍になった。進んだ距離も、3.84 mから6.16 mに伸びた。一方、長さは461ステップ（約3.7秒）で、1000ステップ（8秒）を立ち続けるところには、まだ届かない。
- **伸び方はゆっくりで、行きつ戻りつする**: 28万ステップで1330まで上がった後、30万ステップで1229に下がっている。学習を3倍に延ばしても、結果は1.5倍ほどである。
- **歩けるようになるには**: SB3の公式の学習の設定集（rl-baselines3-zoo。10節）では、Walker2d（v4）のSACは、100万ステップ学習させる設定になっている（学習を始める前に、でたらめな行動で経験をためる `learning_starts` も、既定の100ではなく10000にしている）。作成に使ったPCの速さ（1秒あたり約84ステップ）では、100万ステップに約3時間20分かかる。作成時には、100万ステップまでは確かめていない。二足は、学習に要る経験の量が、車両よりずっと多い。この量の差が、6-2節の「GPUで環境を並べる」につながる。

### 4-4.（任意）動きを見る

学習したモデルを、ウィンドウに表示しながら1エピソード走らせる。WSLgでウィンドウが開く環境（フェーズ1の `turtlesim` のウィンドウが開いた環境）で行う。

ファイル: `~/rl_practice/view_walker.py`

```python
# 学習したWalker2dの方策を、ウィンドウに表示しながら1エピソード走らせる（WSLgのウィンドウが開く）。
import gymnasium as gym
from stable_baselines3 import SAC

env = gym.make('Walker2d-v5', render_mode='human')
model = SAC.load('walker_sac_100000')
obs, info = env.reset(seed=0)
done = False
while not done:
    action, _ = model.predict(obs, deterministic=True)
    obs, reward, terminated, truncated, info = env.step(action)
    done = terminated or truncated
env.close()
```

- `render_mode='human'` を付けると、MuJoCoが、ステップごとにウィンドウへ描く（2節で入った `glfw`・`pyopengl` を使う）。
- `SAC.load` は、4-2節で自分で保存した `.zip` を読む（7-4の2-2節のとおり、他の人から受け取った `.zip` は読み込まない）。

```bash
python view_walker.py
```

**期待する結果**（作成時は、ウィンドウを開かないため実行していない。下は、Gymnasiumの描画の仕組みと、ウィンドウを開かずに同じスクリプトを走らせた結果から想定したもの）: ウィンドウが開き、2本の脚のロボットが、地面の上で前へ進み、約2.4秒で倒れる。倒れるとエピソードが終わり、ウィンドウが閉じる。このスクリプトは種0の1エピソードだけを走らせる。ウィンドウを開かずに同じ種で走らせると、298ステップ（約2.4秒）で3.54 m進んで倒れた（4-2節の表は5つの種の平均なので、値が少し違う）。

## 5. 車両の題材との違い

| | 車両（7-3〜7-5） | 二足（Walker2d） |
|---|---|---|
| 観測 | 5つ（目標・速度・偏差・前回のペダル・次の変わり目までの時間） | 17個（胴体の高さと角度、関節の角度、速度と角速度） |
| 行動 | 1つ（ペダル） | 6つ（関節のトルク） |
| 1ステップ | 0.1秒 | 0.008秒 |
| エピソードの終わり | 90秒たったら（課題の終わり） | 倒れたら（最長8秒） |
| 何もしないと | 抵抗で減速し、やがて止まる | 倒れる |
| 報酬 | 目標との差と、行き過ぎの減点 | 立っている分・前に進んだ分・力の罰 |
| 人が書く制御器 | PI制御（ゲインの2つの数） | 簡単な式では書きにくい |
| 作成時の学習の量と結果 | 約10万ステップで、PI制御に近い追従（7-4） | 10万ステップでは、まだ立ち続けられない（4-2節） |

- **観測と行動が増えると、試す組み合わせが一気に増える**: 車両のNNは、観測5つからペダル1つを決めればよかった。二足では、17個の観測から6つの関節のトルクを同時に決める。どの関節をどれだけ動かせば前に進めるかを、試行錯誤だけで見つけなければならない。
- **倒れると、そこで経験が途切れる**: 車両は、どんなに下手なペダルでも90秒走り、その間の経験がたまった。二足は、学習の始めは0.1〜0.4秒で倒れるので、1エピソードから得られる経験がとても短い。立っていられるようになるまでは、「歩く」ための経験がほとんどたまらない。
- **報酬は、目的を細かく書き分ける**: 「倒れない」だけを報酬にすると、その場で立ち尽くすことを学ぶかもしれない。「前に進む」だけにすると、前へ倒れ込むことを学ぶかもしれない（4-2節の10万ステップは、それに近い）。二足歩行の研究では、足の高さ、胴体の傾き、関節の動きのなめらかさなど、多くの項目を報酬に足して、望む歩き方に近づけることが多い。7-2・7-4で、行き過ぎの重みを変えると学ぶ動きが変わったのと、同じ考え方である。

## 6. 二足歩行に進むために

ここからは読み物である。このページで触れた道具とPCの要件は、2026-10-08に公式の文書で確かめた（10節）。版や要件は変わるので、取り組むときに改めて確かめる。

### 6-1. 足りない知識

車両の題材で学んだこと（ROS2のノード・トピック・launch、物理シミュレータ、記録と指標、強化学習の環境・報酬・学習・評価）は、二足歩行でもそのまま土台になる。そのうえで、次のことが加わる。

- **接触**: 足が地面に着く・離れる・滑る。接触の瞬間に力が急に変わるので、車両の車輪の転がりより、シミュレーションも制御も難しい。
- **バランス**: 重心の位置と、足の裏が地面に着いている範囲の関係で、倒れるかどうかが決まる。歩行では、片足で立つ間も、倒れないように重心を運ぶ。
- **歩き方の型**: 脚をどの順番で、どのくらいの周期で動かすか。強化学習では、報酬や観測にこの型の手がかりを入れることが多い。
- **状態の推定**: シミュレーションでは、胴体の高さや速度を直接読める（このページの観測）。実物のロボットでは、関節の角度のセンサや、加速度と角速度を測るセンサ（IMU）から、推し量る必要がある。
- **シミュレーションと実物の違い**: 7-3の1節で見たとおり、式やシミュレータで学んだものは、違う物理の上ではそのまま通用するとは限らない。二足歩行では、学習中に、摩擦や質量、センサの雑音などをわざとランダムに変え、どれにも耐える方策を学ばせる方法（ドメインランダム化）がよく使われる。

### 6-2. 学習の量と、GPUで環境を並べること

4-2節の学習は、1つの環境を1ステップずつ進め、作成に使ったPC（専用のGPUが無いPC）のCPUで、1秒あたり約84ステップだった。10万ステップで約20分である。

二足やヒューマノイドのロボットの学習では、桁違いの量の経験を使う。Isaac Labの公式の文書（10節）には、強化学習のライブラリの速さを比べた例がある。1台のGPU（NVIDIA GeForce RTX 4090）で、ヒューマノイドの環境（`Isaac-Humanoid-v0`）を4096個並べ、合わせて約6550万ステップを、約200秒（ライブラリによって198〜287秒）で学習させている。同じ量を、4-2節の速さで1つずつ計算すると、 $65500000 \div 84 \approx 780000$ 秒、約9日かかる。

この差を生むのは、**物理の計算を、GPUで、何千もの環境について同時に行う**ことである。7-4の3-1節では、Gazeboの環境は1つのワールドに1つしか作れず、1ステップずつ進めた。GPUで動く物理シミュレータ（Isaac Lab、MuJoCo Playground）は、何千ものロボットを同時に進め、その経験をまとめてNNの学習に使う。強化学習のライブラリも、この並べ方に合わせて作られたものが使われる（6-3節）。

### 6-3. 道具

| 種類 | 道具 | 特徴（公式の文書による） |
|---|---|---|
| 物理シミュレータ | MuJoCo | このページで使ったもの。CPUで動き、`pip` で入る（Apache License 2.0） |
| | MuJoCo Playground | MuJoCoのGPU版（MJX）の上に作られた、GPUで動くロボットの学習の環境集。四足と二足の歩行の環境がある（Apache License 2.0）。NVIDIAのGPUとCUDA 12を前提にした導入の手順になっている |
| | Isaac Lab | NVIDIAのIsaac Simの上に作られた、ロボットの学習の枠組み。四足（Spot、Go2等）とヒューマノイド（Unitree H1・G1）の歩行の環境がある。GPUが必須（6-5節） |
| | Gazebo | フェーズ5〜7で使ったもの。ROS2との連携に強いが、強化学習のために何千も並べる使い方は想定していない |
| 強化学習のライブラリ | Stable-Baselines3 | このページまで使ったもの。1つの環境で動かして学ぶのに分かりやすい。Isaac Labの比較の表では、環境を並べた学習（vectorized training）の欄が「No」になっている |
| | rsl_rl | 脚のロボットの学習で使われるライブラリ（BSD-3-Clause）。PPOなどを持ち、Isaac Labや、MuJoCo Playgroundなどで使われている |
| | skrl | PyTorchとJAXで動く、部品を組み合わせて使うライブラリ（MIT License）。Gymnasium、Isaac Lab、MuJoCo Playgroundなどの環境に対応する |

学習計画（[`docs/learning_plan.md`](learning_plan.md)）では、二足歩行の段階で、SB3からskrl（またはrsl_rl）へ移ることにしている。Isaac Labは、4つのライブラリ（skrl・rsl_rl・RL-Games・SB3）のどれでも学習させられるので、まず慣れたSB3で動かし、並べた学習の速さが要るところで、ほかのライブラリへ移ることもできる。

### 6-4. ROS2の側で要るもの

学習した方策を、ROS2でロボット（シミュレータか実物）につなぐには、7-5と同じ考え方（重みを読むノードを、制御器の代わりに置く）に加えて、次の部品が要る。どれも、このプロジェクトではまだ扱っていない。

- **URDF**: ロボットの形（リンクと関節、質量、見た目）を書くファイルの形式。脚の長さや関節の範囲を、ROS2とシミュレータで共有する。
- **tf2**: ロボットの各部分（胴体、脚、足）の位置と向きの関係を、時刻つきで管理する仕組み。関節の角度から、足先がどこにあるかを求めるのに使う。
- **ros2_control**: 関節のモータに指令を送り、状態を読む部分を、決まった形で書くための枠組み。シミュレータと実物で、同じ制御のノードを使い回しやすくなる。

アイデアの元（[`docs/idea_origin.md`](idea_origin.md) のステップ7）では、これらを、車両の簡単なURDFから練習することを候補に挙げている。

### 6-5. PCの性能

Isaac Labの今の安定版の導入の案内（10節。2026-10-08に確認）は、次を求めている。

- OS: Ubuntu 22.04（Linux x64）またはWindows 11（x64）
- メモリ（RAM）: 32 GB以上
- GPUのメモリ（VRAM）: 16 GB以上
- Isaac Sim 5.1（Pythonは3.11）

このプロジェクトのROS2（Jazzy）は、Ubuntu 24.04・Python 3.12で動いているので、Isaac Labは、ROS2とは別の環境に入れることになる。MuJoCo Playgroundも、NVIDIAのGPUを前提にしている。専用のGPUが無いPCでは、このページのように、CPUのMuJoCoで小さく試すところまでが現実的である。本格的な二足歩行の学習は、性能の高いPCで行う（アイデアの元の、「より性能の高いPCを使えるようになってから、または別プロジェクトとして」という方針）。

## 7. 本フェーズのまとめ

- MuJoCoの2次元の二足（Walker2d）を、車両と同じSB3のSACで学習させた。観測17・行動6になっても、書き方はほとんど変わらない。
- 二足は、何もしないと倒れる。倒れるとエピソードが終わるので、学習の始めは経験がとても短い。10万ステップ（作成に使ったPCで約20分）では、まだ立ち続けられず、前へ倒れ込むように進むところまでだった。
- 報酬は「倒れない」「前に進む」「力を使いすぎない」を同時に求める。何を報酬に入れるかで、学ぶ動き方が変わる。
- 本格的な二足歩行の学習では、GPUで何千もの環境を並べ、桁違いの量の経験を使う（Isaac Labの例で、約6550万ステップを、最も速いライブラリで約200秒）。そのための道具（Isaac Lab・MuJoCo Playground、rsl_rl・skrl）と、ROS2の側の部品（URDF・tf2・ros2_control）、要るPCの性能を整理した。

## 8. つまずきやすい点

| 症状 | 確認すること |
|---|---|
| `gym.make('Walker2d-v5')` で `DependencyNotInstalled`（MuJoCoが無い）と出る | 2節の `pip install "gymnasium[mujoco]==1.3.0"` を、強化学習の仮想環境を有効にしたターミナルで実行したか |
| `ModuleNotFoundError: No module named 'gymnasium'` 等 | 仮想環境を有効にしたか（`source ~/rl_venv/bin/activate`） |
| 4-2節の値が、期待する結果と違う | 学習は、PCや版、計算に使うスレッドの数で変わる（7-4の6節）。収益と長さがおおむね増えていれば、学習は進んでいる |
| 4-4節の任意の「動きを見る」でウィンドウが開かない、または `glfw` のエラーが出る | WSLgで画面を出せる環境か（フェーズ1の `turtlesim` のウィンドウが開いたか）。開かない場合は、このページの数値で読み取る |
| 学習に時間がかかりすぎる | 4-2節は約20分かかる。途中で止める場合は `Ctrl+C`（モデルは保存されない）。`--steps 40000` のように短くしてもよい（2万の倍数で指定する） |

## 9. 次へ

フェーズ7の7-0〜7-6で、強化学習の基本から、車両のゲインの学習、NNの方策の学習とROS2のノードへの持ち出し、二足歩行への橋渡しまでを一通り行った。このプロジェクトのフェーズ8（学習計画で保留中）や、二足歩行に本格的に取り組む次のプロジェクトでは、6節で整理したことが出発点になる。

## 10. 公式ドキュメント・参考資料

確認状況（2026-10-08）: 下のページは、実在を確認した（HTTP 200）。Walker2dの観測・行動・報酬・終わり方・1ステップの長さは、Gymnasiumの公式の文書で、`[mujoco]` が求める版は、Gymnasium 1.3.0のパッケージの情報で確かめた。rl-baselines3-zoo のWalker2dのSACの設定は、その設定ファイルで、Isaac Labの要件と、ライブラリの比較（GPU・環境の数・学習の時間・機能の表）は、Isaac Labの公式の文書で、MuJoCo Playground・rsl_rl・skrlの特徴とライセンスは、それぞれの公式のリポジトリかPyPIのページで確かめた。

### 公式

- [Gymnasium — Walker2d](https://gymnasium.farama.org/environments/mujoco/walker2d/)（観測・行動・報酬・終わり方）
- [MuJoCo — Overview](https://mujoco.readthedocs.io/en/stable/overview.html)（MuJoCoの概要）
- [PyPI — mujoco](https://pypi.org/project/mujoco/)（MuJoCoのPythonのパッケージ）
- [Isaac Lab — Installation](https://isaac-sim.github.io/IsaacLab/main/source/setup/installation/index.html)（PCの要件）
- [Isaac Lab — Reinforcement Learning Library Comparison](https://isaac-sim.github.io/IsaacLab/main/source/overview/reinforcement-learning/rl_frameworks.html)（ライブラリの比較）
- [MuJoCo Playground（GitHub）](https://github.com/google-deepmind/mujoco_playground)
- [rsl_rl（GitHub）](https://github.com/leggedrobotics/rsl_rl)
- [rl-baselines3-zoo — SACの設定（`hyperparams/sac.yml`）](https://github.com/DLR-RM/rl-baselines3-zoo/blob/master/hyperparams/sac.yml)（SB3の公式の学習の設定集。Walker2dの学習のステップ数）
- [skrl（GitHub）](https://github.com/Toni-SM/skrl)
- [ROS 2 Documentation (Jazzy) — URDF](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/URDF-Main.html)
- [ROS 2 Documentation (Jazzy) — Tf2](https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Tf2-Main.html)
- [ros2_control documentation — Jazzy](https://control.ros.org/jazzy/index.html)

> 出典: Walker2d・MuJoCo・Isaac Lab・MuJoCo Playground・rsl_rl・skrl・ROS 2の説明は、各公式の文書を自分の言葉で要約したもので、逐語の転載ではない（ROS 2 Documentation は CC BY 4.0）。6-2節の学習の時間の数は、Isaac Labの公式の文書の比較の例から引いた。このページのサンプルコード（`check_walker.py`・`train_walker.py`・`view_walker.py`）は独自に書いたもので、7-0・7-2のサンプルの形を元にしている。
