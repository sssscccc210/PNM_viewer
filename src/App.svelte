<script>
  import Viewer from "./lib/Viewer.svelte";
</script>

<main>
  <h1>PNMビューワー</h1>
  <hr>
  <Viewer />
  <hr>
  <p>
    PNMファイルの仕様（本家）は，<a href="https://netpbm.sourceforge.net/doc/index.html">https://netpbm.sourceforge.net/doc/index.html</a>。<br>
    英語を読みたくないよ！って方は，<a href="https://coskxlabsite.stars.ne.jp/html/for_students/PNM/pnmfiles.html">https://coskxlabsite.stars.ne.jp/html/for_students/PNM/pnmfiles.html</a>
    がおすすめです。<br><br>
    現在，ファイルが「ASCIIタイプ（P1〜P3）でデータが正常」という範囲で正常に動作します（仕様違反検知等は開発中です）。
  </p>
</main>