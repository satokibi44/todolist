TODOList
====
# 1. はじめに
## 1.1. 概要
今回は某インターンの課題としてTODOリストを作成した.<br>
TODOリストの主な仕様としては,<br>
TODOリストの追加(追加時間,TODO名,〆切日をデータベースに保存する)<br>
TODOリストの編集(データベース上のTODO名と〆切日の変更)<br>
TODOリストの検索(データベース上のTODO名の検索)<br>
である.
また開発環境はEclipse,サーバーはAWSを使用した.
## 1.2. 使用した技術要素
使用言語...JAVA<br>
フレームワーク...spring boot<br>
サーバー...AWS EC2<br>
データベース...MySQL<br>
## 1.3.テスト環境
safari
# 2.開発環境のセットアップ手順
## 2.1.雛形の作成方法
ソースコードの雛形は Spring Initializr(https://start.spring.io
)で作成し,最初のプルダウンはGradle を指定した.
またSearch for dependencies には Web, JPA, Thymeleaf, DevTools, MySQL を指定した.
![suteru_fay](https://user-images.githubusercontent.com/52820882/62268237-2c0b9380-b46a-11e9-9b95-aaaeb74bf46d.png)
## 2.2.Eclipseのセットアップ
今回は日本語プラグインは用いず英語でセットアップした.
まずEclipseのダウンロードページ（https://www.eclipse.org/downloads/packages/
）
から一番上のインストーラをダウンロードする.インストールが終わると早速起動し,HELPバーにあるEclipse Market PlaceからBuildship Gradle Integrationをダウンロードする.
![suteru_fay](https://user-images.githubusercontent.com/52820882/62268807-0089a880-b46c-11e9-8756-80f547b7ef65.png)
![suteru_fay](https://user-images.githubusercontent.com/52820882/62274352-ae9c4f00-b47a-11e9-8d49-d865daddf413.png)
EClipseの環境設定が完了するとEclipse内でSpring bootのプロジェクトが実行&ビルドできるので便利である.
## 2.3.AWSのセットアップ
AWSのアカウントはすでに持っている程で解説する.<br>
まずサービスバーからコンピューティング->EC2に移動する.<br>
するとインスタンス作成というボタンが中央付近に現れるのでそれをクリックする.<br>
AMIにはAmazon Linux AMI 2018.03.0 (HVM), SSD Volume Type<br>
インスタンスタイプはt2.microe<br>
セキュリティグループにはssh,http,またカスタムTCPルールとしてポート番号8080を追加する.<br>
![suteru_fay](https://user-images.githubusercontent.com/52820882/62267104-ffee1380-b465-11e9-8c42-96ae5ec9ce06.png)
以上のセットアップをしサーバーを起動する.この際に出てくるpemファイルはわかりやすい場所に保存する.パブリックDNSも大切なのでコピーして保存されたい.<br>
![suteru_fay](https://user-images.githubusercontent.com/52820882/62268885-4d6d7f00-b46c-11e9-90ca-1d7db2fea4cc.png)

次にEC2にssh接続する.コマンドウィンドウを開いて
```
$ssh -i [pemファイルの場所/~.pem] ec2-user@[自分のパブリックDNS]
$exit
```
接続が終わり初期設定も終わると$exitでログアウトしておく.次にspring bootで作ったプロジェクトを(https://confrage.jp/stsgradleで作成したspring-bootの実行可能jarを作成する方法/
)を参照しjarファイルを作ってec2インスタンス上に配置する.転送前にjarファイルのあるディレクトリにcdを使って移動する必要があるので注意されたい.以下がec2上に転送するための方法だ.転送が終わるとログアウトする.
```
$ sftp -i [pemファイルの場所/~.pem] ec2-user@[パブリックDNS]
$ sftp> put [jarファイルの名前.jar]
$ exit
```
次にDBのセットアップを行う,通常AWSでDBを使う場合EC2+RDSが一般だと思うが今回はEC2内にMySQLをインストールしてEC2内で完結させた.
以下のようにセットアップを行なった.
```
$sudo yum install  mysql56 mysql56-server //MySQLのインストール
$sudo service mysqld start//MySQLの起動
$mysql -u root //MySQLへログイン
```
セットアップが完了しMySQLへログインできると,後は自分のspring bootプロジェクトに合わせてMySQLを構築していく(Create Databaseやcreate tableなど...)私の場合は以下のように構築した.
```
mysql> create database ASA;
mysql> use ASA;
mysql> CREATE TABLE `test3` (
      `listno` bigint(20) NOT NULL AUTO_INCREMENT,
      `todoname` varchar(31) NOT NULL DEFAULT "TODO",
      `untildate` varchar(10) NOT NULL DEFAULT "2018-01-01",
      `createdate` varchar(10) NOT NULL DEFAULT "2018-01-01",
      `complete` boolean NOT NULL DEFAULT 0 ,
      `color` varchar(10) NOT NULL DEFAULT "red",
      PRIMARY KEY (`listno`)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8
```
DB構築が終わるとjarが実行可能になるので以下のように実行する.
```
$ssh -i [pemファイルの場所/~.pem] ec2-user@[自分のパブリックDNS]//サーバーへssh接続
$java -jar [jarファイルの名前.jar] 
```
# 3. 設計・構成についての説明
主にこのWebアプリは3つの画面から構成される.<br>
1.TODO追加画面<br>
2.TODO編集画面<br>
3.TODO検索画面<br>
である.これらの構成の説明の前にデータベースで設定したカラムについて説明する.<br>
## 3.1.設定したカラムについての説明
今回設定したカラムは,TODO名を表すtodoname,idを表すlistno,締め切り時間を表すuntildate,TODOを作った時間のcreatedate,完了状態を表すcompleteとcolorである.完了パラメータをcompleteとcolorに分けた理由であるがボタンを押した時に動作として背景の変更と文字の変更の２種類が求められたためである文字の変更のパラメータをcomplete,背景の変更のパラメータをcolorとした.もっと良い方法があったと思うが実装の方法がわからなかったのでこの仕様にした.未完了を表す値はcompleteで0,colorでred.完了を表す値はcompleteで1,colorでblueである.デフォルトは未完了状態である.
![suteru_fay](https://user-images.githubusercontent.com/52820882/62187765-5f381f00-b3a5-11e9-92ac-52f73f0ae60e.png)
どうやってcompleteとcolorの値でボタンを制御したかであるが,以下のようにパラメータの値をhtmlに変換し文字の方はvalueに背景の方はstyleに指定した.
```java:Employee.java
public String getComplete() {
    if(complete!=1) {
    	return("未完了");
    }
    return("完了");
}
public String getColor() {
    if(!(color.equals("blue"))) {
    	return("background-color:salmon;");
    }
    	return("background-color:blue;");
}
```
## 3.2.TODO追加画面
TODO追加の際の転送するパラメータについては画面上に入力されたTODO名と締め切り時間,またTODOが未完了である事を認識させるため数字デフォルトで0を,ボタンの実装に必要なデフォルトの値"red"を追加ボタンを押すとサーバーに転送する.つまりTODO追加を行うとカラムcolorとcompleteに対しては未完了の初期化が行われる.MySQLでtableを作る際にdafaultでcomplete=0とcolor=redを設定したがなぜか当プロジェクトからリクエストを送るとnullとして認識されたためこのような仕様にした.以下に
todo追加の際のサンプルプログラムを示す.ここでのnとはEmployeeクラスのインスタンス,empRepositoryとはEmployeeRepositoryクラスのインスタンスのことである.
```java:HelloController.java
n.setTodoname(text1);//toDO名の追加
n.setUntildate(Date);//締め切り時間の追加
n.setColor("red");//ボタンの背景のための未完了フラグの追加
Long i =(long) 0;
n.setComplete(i);//ボタンの文字のための未完了フラグの追加
LocalDate d = LocalDate.now();
DateTimeFormatter df1 = 
DateTimeFormatter.ofPattern("yyyy-MM-dd");
n.setCreatedate(df1.format(d));//追加時間の追加
empRepository.save(n);
```
完了ボタンを押した時の挙動としては,押すまえの状態が完了状態だとボタンの文字を未完了,背景を赤に未完了状態だとボタンの文字を完了,背景を青にデータベース上でupdateする.プログラムは以下の通りである.<br>
HelloController.java
```java:HelloController.java
if(empRepository.findComp(colorid).equals("blue")) {//ボタンを押し前は完了状態か？
  empRepository.update2(0,"red",colorid);//ボタンを押す前が完了状態だった時のupdate
 }else {
  empRepository.update2(1,"blue",colorid);//ボタンを押す前が未完了状態だった時のupdate
 }
 List<Employee> emplist=empRepository.findAll(new Sort(Sort.Direction.DESC,"id"));//TODOが新しい順に並び替え
 model.addAttribute("employeelist", emplist);
 return "index";
```
MySQLでのアップデートの実装は以下である.<br>
EmployeeRepository.java
```java:EmployeeRepository.java
@Modifying
@Transactional
@Query(value = "update test3 t set"
		+ " t.complete = :complete,"
		+ " t.color = :color"
		+ " where t.listno = :listno", nativeQuery = true)
public int update2(
	@Param("complete") int complete,
    	@Param("color") String color,
    	@Param("listno") int listno);
```
上のjava:EmployeeRepository.javaを見てもわかるようにupdateはwhereで条件指定したかったのでクエリで実装した.また3.4節で述べる検索画面での検索の実装や3.3節で述べる編集画面の編集すべきTODOの表示などもselectする時に条件指定する必要があったのでクエリで実装した.詳しくはEmployeeRepository.javaを参照されたい.クエリで実装したためセキュリティ面が弱いのではないかとも考えた.
次に編集ボタンの説明をする.編集ボタンについてはTODOのidで編集すべきTODOを認識して編集する.HelloControllerクラスのchangeメソッドがそれに当たる.
## 3.3.TODO編集画面
編集画面についてはTODO追加画面,または検索画面でのボタンが押されたTODOのIDを取得して,IDに応じたTODO名と締め切り時間の表示をする.
編集画面で編集事項が入力されボタンを押されるとTODOがアップデートされる.HelloControllerクラスのchメソッドでボタンを押したidに対応したtodoデータ(todo名,締め切り時間,作成時間,完了フラグ２個)をchange.htmlに表示する.
```java:EmployeeRepository.java
@RequestMapping(value = "change",method = RequestMethod.GET)
   public String change(@RequestParam("id")Integer id,
		Model model) {
   List<Employee> emplist=empRepository.findById1(id);
   model.addAttribute("employeelist", emplist);
   nowId=id;
   return "change";
	}
```
以下の図は編集画面に関するフローである.
![suteru_fay](https://user-images.githubusercontent.com/52820882/62311116-07450980-b4c6-11e9-9906-e873a140e3dc.png)
また実際に変更した際の動作はchangeメソッドであるが,仕様は3.2節のTODO追加とほぼ変わらないので省略する.
## 3.4.TODO検索画面
検索画面においては,未完了状態のTODOしか検索できない仕様である.<br>
検索についてはTODO名の部分一致のものを検索するのが要求仕様であったため,クエリを以下のようにLIKE詞で修飾した.<br>
```mysql:sample.sql
select * from test3 where todoname like ?1% AND complete=0
```
検索結果で出てくる編集ボタンや完了ボタンの挙動については,3.2節のTODO追加画面と同じである.HelloControllerクラスのsearchメソッドが検索画面の実装部分になる.
## 3.4.エスケープ処理について
Spring bootではjavaプロジェクトからhtmlに値を渡す時にth:textなどth:~で指定するがこの時にデフォルトでエスケープ処理が行われる.<br>
なおエスケープ処理を行いたくない場合はth:utextで値を渡してやると良い.<br>
デフォルトでエスケープが行われることを知らずハマった箇所である.
![suteru_fay](https://user-images.githubusercontent.com/52820882/62346076-c386ea00-b52f-11e9-9f76-3958abfc6e02.png)
# 最後に
## 今後の展望
今回はEC2の中にMySQLを実装したので次はEC2とRDSのサブシステムに分けてインフラ構築したい.<br>
また今回はクエリで条件付きのSQL文を実装したがセキュリティ面が心配である.今後の展望としてはクエリを仕様しない条件付きSQL文の実装をしたい.
## ハマった箇所
プロジェクトを実際に公開する際にssh接続して$java -jar [jarファイルの名前.jar] で実行すると書いたがこれでは一定時間経つとspring bootがシャットダウンしてしまうようだ.<br>
代わりに
```
$ nohup java -jar [jarファイルの名前.jar].jar > output.log 2>&1&
```
で実行するとシャットダウンせずに公開することができた.
