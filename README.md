# csv2urdf
csvのリンク／関節リストからURDFファイルを生成するツールです．
ROS1用は下記リンクページで公開しています．<br>
https://sites.google.com/site/robotlabo/ros/%E3%83%AD%E3%83%9C%E3%83%83%E3%83%88%E3%83%A2%E3%83%87%E3%83%AB%E4%BD%9C%E6%88%90%E3%83%84%E3%83%BC%E3%83%AB
<br>
ROS1用のプログラムはPython2で動作可能なことは確認しています．

こちらのパッケージはPython3に対応するように修正したものです．

## クイックスタート
テンプレートファイルからURDFファイルを生成するために<b>jinja2</b>を利用しています．

jinja2のインストールはコマンドプロンプトで下記の通り入力すればOK.

    pip3 install jinja2
pipのコマンドが見つからないときは下記コマンドでインストールしておく．

    sudo apt install python3-pip



## はじめに
ロボットのモデルを<a href="http://wiki.ros.org/urdf">URDF</a>で簡単に記載できますが，間違えやすいので表を使ってリンクごとのパラメータを一括して入力することで全体のURDFモデルを出力するpythonのプログラムを作りました．

　URDFでは，リンクの親子関係を接続する関節（ジョイント）とリンクの位置と大きさを入力していくことで木（ツリー）構造でつながりを表現していきます．<br>
　この1つのリンクに複数のジョイントを繋げた状態を表す表を作成します．<br>
その表を読み込んで，モデル生成のツールでURDFファイルを生成します．

URDFの基本のリンク記載例を確認します．
１つのリンクの記載の例は下記の通りです．

　リンクの記載例：

    <!-- Link 1 -->
    <link name="link1"> #リンクのパラメータ設定開始 nameでリンクの名前指定
        <collision> #collisionで接触判定を行うリンクのサイズ指定
        <origin rpy="0 0 0" xyz="0 0 0.075"/>  #サイズ指定を行うときの原点座標の位置(xyz)と姿勢(rpy)を固定角で指定
        <geometry>
            <box size="0.05 0.05 0.15"/> #原点座標の位置・姿勢を中心に直方体のサイズ(x,y,z)を指定
        </geometry>
        </collision> #接触判定の記載はここまで
        <visual> #rvizなどで表示するリンクのパラメータ：普通はcollisionと同じ
        <origin rpy="0 0 0" xyz="0 0 0.075"/>
        <geometry>
            <box size="0.05 0.05 0.15"/>
        </geometry>
        <material name="red"/> #collisionと違うのは色指定があること
        </visual> #リンクの描画はここまで
        <inertial> #リンクの質量や慣性モーメントを定義
        <origin rpy="0 0 0" xyz="0 0 0.075"/> #リンクの質量中心位置と座標の姿勢を指定
        <mass value="1"/> #リンクの質量を指定
        <inertia ixx="1.0" ixy="0.0" ixz="0.0" iyy="1.0" iyz="0.0" izz="1.0"/> #リンク座標における慣性モーメントのパラメータ指定
        </inertial>
    </link> #リンクのパラメータ設定ここまで

　関節の記載例：

    <joint name="joint1" type="revolute"> #関節のパラメータ記載開始 nameで名前を指定，typeで回転"revolute"か固定"fix"を指定
        <parent link="link1"/> #ジョイントが繋ぐ親のリンク（接続元）の名前を指定
        <child link="link2"/>#ジョイントが繋ぐ子のリンク（接続先）の名前を指定
        <origin rpy="0 0 0" xyz="0 0 0.15"/> #ジョイントを指定する座標系の位置・姿勢を指定
        <axis xyz="0 0 1"/> #指定した座標系のどこに関節軸があるか指定
        <limit effort="30" lower="-2.617" upper="2.617" velocity="1.571"/> #関節のトルク(effort)，関節角(rad指定）の下限(lower)，関節角の上限(upper)を指定
    </joint> #関節のパラメータ記載修了

上記のリンクと関節のパラメータ設定例から下記のパラメータを表で管理することとする．

### パラメータ
- link リンク名
- child_link リンク(link)の子供のリンク名：ジョイント作成時に現在のリンクと接続するリンクの名前．複数あるときは「 , 」区切りでスペースなしで記載．(ex.link1,link2,link3)
- link_rpy リンクの姿勢をxyz固定角(roll, pitch, yaw）で指定
- link_xyz リンクの原点位置をxyz座標系で指定（ただし，rpyでの回転前の座標系で）
- link_size リンクの原点位置を中心に直方体のサイズを（x y z）で指定．（こちらはrpyで回転後の座標系）
- link_color リンクのカラー指定．（今回はテンプレートにデフォルトのカラーを記載することにする）
- link_mass リンクの質量
- ink_ixx リンクの回転モーメント xx成分（以下略）
- link_ixy
- link_ixz
- link_iyy
- link_iyz
- link_izz
- joint linkとchild_linkを繋ぐ関節の名前（ただし，今回はジョイントの名前は自動的に決定することにする）
- joint_type* 関節の種類（fix：固定，revolute：回転で指定）
- joint_rpy* 関節の姿勢をxyz固定角(roll, pitch, yaw）で指定
- joint_xyz* 関節の原点位置をxyz座標系で指定（ただし，rpyでの回転前の座標系で）
- joint_axis* 関節軸の方向を指定（x軸に回転軸があるときは(1 0 0) )
- joint_effor*t 関節の最大トルク
- joint_lower* 関節が動くことのできる最小角度
- joint_upper* 関節が動くことのできる最大角度
- joint_velocity* 関節角の最大速

*には子供の数だけ複数定義する．記載方法は例を参照．


下図のような３自由度のマニピュレータを構成したいときを考える．

![alt text](image.png)

上記ロボットのパラメータの表をcsvファイルで記載した例を示す．
| link | child_link | link_rpy | link_xyz | link_size | link_color | link_mass | ... |
|------|------------|----------|----------|-----------|------------|-----------|-----|
| world | base_link | 0 0 0 | 0 0 0 | 0 0 0 | gray | 0.001 | ... |
| base_link | Link1 | 0 0 0 | 0 0 0 | 0.5 0.5 0 | blue | 0.001 | ... |
| Link1 | Link2 | 0 0 0 | 0 0 0.05 | 0.05 0.05 0.1 | red | 1 | ... |
| Link2 | Link3 | 0 0 0 | 0 0 0.05 | 0.05 0.05 0.1 | green | 1 | ... |
| Link3 | Link4 | 0 0 0 | 0 0 0.05 | 0.05 0.05 0.1 | blue | 1 | ... |
| Link4 | 0 | 0 0 0 | 0 0 0.05 | 0.05 0.05 0.1 | red | 1 | ... |

詳しくは，csvディレクトリ内にある3dofarm.csvファイルを確認してください．


## URDFの生成
csvファイルを読み込んで，URDFの基本の型をテンプレートファイルとして用意したものに流し込んで，テンプレートファイルから目的とするurdfファイルを生成します．

上記作業には，jinja2を利用するのでインストールしていない人は最初の方へ戻ってjinja2をインストールをしてください．

pythonでcsvファイル読み取りからurdfファイルを生成するコードは，<a href="./create_robot_csv.py">create_robot_csv.py</a>になります．
URDFのテンプレートファイルは<a href="./template.urdf">template.urdf</a>です．

２つのファイルを使って3dofarm.csvからurdfを生成するときのコマンドは下記．

    python create_robot_csv.py 3dofarm.csv
    6 links data read  <-というメッセージが出るはず．

3dofarm.csvがおいてある同じディレクトリ内に3dofarm.urdfが生成されているはずです．

下記コマンドで，問題なくurdfファイルが生成されているか確認します．

    check_urdf 3dofarm.urdf <-このコマンドを打つと下記のように表示されるはず
    robot name is: 3dofarm
    ---------- Successfully Parsed XML ---------------
    root Link: world has 1 child(ren)
        child(1):  base_link
            child(1):  Link1
                child(1):  Link2
                    child(1):  Link3
                        child(1):  Link4


### 挙動についての注意点
- 指定したcsvのファイル名（パスを含む）がrobotnameに自動で登録されるので，create_robot_csv.pyと3dofarm.csvは同じディレクトリで実行してください．
- リンクとジョイントのテンプレートに数値が流し込まれていく課程でif文で条件を判断してテンプレートと実際に必要な部分との整合性を取っていきます．
- jointの名前は自分でつけたいかもしれませんが，今回のテンプレートでは親リンクの名前と子供のリンクの名前をつなげた名前を自動的に割り振っています．
- ex. Link1とLink2の関節名：Link1_2_Link2
- IGN Gazeboで利用したいときの注意点：リンクサイズのx,y,zのいずれか一つでもゼロが入っているとエラーで生成されないので，小さい数字を必ず入れておいてください．


## XACROの利用
xacroを利用するときは，先程用意したテンプレート(template.urdf)の最初にある"robot name"を指定する場所の後ろにxacroを利用する宣言"xmlns:xacro="http://www.ros.org/wiki/xacro"を記載するだけで利用できるようになります．

xacroファイルを作るときのテンプレートは<a href="./template.xacro">template.xacro</a>です．<br>

templete.xacroを読み込んで.xacroファイルを出力するためのpythonのスクリプトは<a href="./create_robot_xacro.py">create_robot_xacro.py</a>です．

２足歩行ロボットのモデル生成の例を下記に示す．

    python create_robot_xacro.py bipedrobot_xacro.csv

上記実行後にcsvファイルのあるディレクトリにbipedrobot_xacro.xacroが生成されているはずです．<br>
このxzcroファイルをurdfに変換したいときは下記コマンドで変換．

    ros2 run xacro xacro bipedrobot_xacro.xacro > bipedrobot_xacro.urdf
    check_urdf bipedrobot_xacro.urdf  #URDFのチェックコマンド


## XACROファイルからURDFまでの一括生成
手動のコマンドでxacroからURDFファイルを生成するのが面倒な場合は<a href="./create_robot_gazebo_xacro.py">create_robot_gazebo_xacro.py</a>を利用してください．<br>
下記利用例です．

    python create_robot_gazebo_xacro.py asterisk_sixdof_r2.csv

上記実行すると，csvファイルと同じ名前のxacroファイル(asterisk_sixdof_r2.xacro)とurdfファイル(asterisk_sixdof_r2.urdf)が生成されていることが確認できる．

