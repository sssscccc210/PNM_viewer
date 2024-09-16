<script>
  const pnmExt = {
    PBM: "pbm",
    PGM: "pgm",
    PPM: "ppm"
  };
  const magicNumbers = {
    PBM: "P1",
    PGM: "P2",
    PPM: "P3"
  }
  let isHover = false;
  let isContentOpen = false;
  let file;
  let content;
  let imgContent;

  let magicNum, width, height, maxBrightness;

  function drawPbm(d){
    let data = [];
    let index = 3;

    if(d[3].length > 1){
      for(let i=3; i<d.length; ++i){
        data.push.apply(data, d[i].split(''));// 参考：https://qiita.com/jkr_2255/items/11e80010602d3387533b
        //console.log(d[i].split(''));
      }
      //console.log(data);
    }
    else data = d;

    imgContent = "<table border='0' style='border-collapse:collapse; border-color:red'>";
    for(let h=0; h<height; ++h){
      imgContent += "<tr>";
      for(let w=0; w<width; ++w){
        imgContent += `<td style='width:1px; height:1px; padding:0; background: rgb(${(data[index]-1) * -255}, ${(data[index]-1) * -255}, ${(data[index]-1) * -255})'></td>`;
        index++;
      }
      imgContent += "</tr>";
    }
    imgContent += "</table>";
  }

  function drawPgm(data){
    let index = 4;

    imgContent = "<table border='0' style='border-collapse:collapse; border-color:red'>";
    for(let h=0; h<height; ++h){
      imgContent += "<tr>";
      for(let w=0; w<width; ++w){
        imgContent += `<td style='width:1px; height:1px; padding:0; background: rgb(${data[index]}, ${data[index]}, ${data[index]})'></td>`;
        index++;
      }
      imgContent += "</tr>";
    }
    imgContent += "</table>";
  }

  function drawPpm(data){
    let index = 4;

    imgContent = "<table border='0' style='border-collapse:collapse; border-color:red'>";
    for(let h=0; h<height; ++h){
      imgContent += "<tr>";
      for(let w=0; w<width; ++w){
        imgContent += `<td style='width:1px; height:1px; padding:0; background: rgb(${data[index]}, ${data[index+1]}, ${data[index+2]})'></td>`;
        index += 3;
      }
      imgContent += "</tr>";
    }
    imgContent += "</table>";
  }
  
  // ドラッグ＆ドロップエリアにイベントを追加
  function handleDragOver(event) {
      event.preventDefault();
      isHover = true;
  }

  function handleDragLeave(event) {
    isHover = false;
  }

  function handleDrop(event) {
      event.preventDefault();
      isHover = false;
      imgContent = "処理中...";
      
      file = event.dataTransfer.files[0];
      if (file) {
        /*let ext = file.name.slice(-3);

        if(!Object.values(pnmExt).includes(ext)){
          imgContent = "PNMファイルのみ対応";
          return;
        }*/

        // PNMファイルを読み込む
        const reader = new FileReader();
        reader.readAsText(file);
        reader.onload = (e) => {
          let noCommentData = e.target.result.toString() // 読み込んだデータをstringに変換
            .replaceAll(/\#.*?(\n|\r)/g, '') // #.....[CR or LF]（コメント部）を削除
            .split(/\s/) // 区切り文字を基ににsplit
            .filter(a=>a!=''); // 空要素を削除
          content = e.target.result.toString().replaceAll('\n', '<br>');// 中身そのまま表示用

          //console.log(noCommentData);
          magicNum = noCommentData[0];
          width = parseInt(noCommentData[1]);
          height = parseInt(noCommentData[2]);
          if(magicNum !== magicNumbers.PBM) maxBrightness = parseInt(noCommentData[3]);

          if(magicNum === magicNumbers.PBM) drawPbm(noCommentData);
          else if(magicNum === magicNumbers.PGM) drawPgm(noCommentData);
          else if(magicNum === magicNumbers.PPM) drawPpm(noCommentData);
          else imgContent = "マジックナンバーが不正です。";

        };
      }
      else{
        imgContent = "ファイルをアップロードしてください。";
      }
  }
</script>

<body>
  <p class="img-info">
    {magicNum || ""}
    {width || ""}
    {height || ""}
    {magicNum !== magicNumbers.PBM ? (maxBrightness || '') : ''}
  </p>
  
  <!-- svelte-ignore a11y-no-static-element-interactions -->
  <div class="drop-area"
    on:dragover={handleDragOver}
    on:dragleave={handleDragLeave}
    on:drop={handleDrop}
    class:hover={isHover}
  >
  {#if imgContent}
    {@html imgContent}
  {:else}
    ここにファイルをドラッグ＆ドロップ
  {/if}
    
  </div>

  <div class="file-info">
    ファイル名：{file?.name || '---'}，サイズ：{(file?.size / 1024).toFixed(2)} KB
    <br>【ファイルの中身】
    <div style="padding-left: 20px">
      {#if content}
        {#if isContentOpen}
          <button on:click={()=>{isContentOpen = false}}>閉じる（軽くなるかも？）</button>
          <p style="font-family: 'Courier New', Consolas, monospace, Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;">{@html content}</p>
        {:else}
          <button on:click={()=>{isContentOpen = true}}>中身を表示</button>
        {/if}
      {/if}
    </div>
  </div>
</body>

<style>
  .drop-area {
      width: fit-content;
      min-width: 400px;
      height: fit-content;
      min-height: 400px;
      border: 2px dashed #ccc;
      border-radius: 10px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 16px;
      color: #888;
      margin: 20px auto;
      text-align: center;
  }
  .hover {
      border-color: #333;
      color: #333;
      background-color: paleturquoise;
  }
  .file-info {
      margin-top: 10px;
      font-size: 14px;
      color: #333;
  }
  .img-info {
      margin-top: 10px;
      font-size: 14px;
      color: #333;
  }
</style>