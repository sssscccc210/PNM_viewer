<script>
    const pnmExt = {
        PBM: "pbm",
        PGM: "pgm",
        PPM: "ppm",
    };
    const magicNumbers = {
        PBM_A: "P1",
        PGM_A: "P2",
        PPM_A: "P3",
        PBM_B: "P4",
        PGM_B: "P5",
        PPM_B: "P6"
    };
    let isHover = false;
    let isContentOpen = false;
    let file;
    let fileName;
    let fileSize;
    let contentStr = ""; // 中身のデータ
    let pos;
    let contentBuffer;// 中身のunit8バイトデータ配列
    let dropAreaObj; // D&D area obj
    let canvasObj; // canvasObj
    let canvasCtx; // context of canvasObj
    let isCanvasHidden = true;
    let message = "ここにファイルをドラッグ＆ドロップ";
    let brightnessScale; // maxBrightnessを基準にした255に対する倍率
    let bytePerPixel;// PGM, PPMの各ピクセル値のサイズ(1 or 2)
    let magicNum, width, height, maxBrightness;
    let tmp;

    const checkBlank = (a)=> ([9, 10, 13, 32].indexOf(a) > -1) ? true : false;// 与えられたバイトの中身（10進数）が区切り文字かを判定する [\t, \n, \r, SP]
    const skipBlank = ()=>{ while( checkBlank(contentBuffer[pos]) ) pos++; };// 区切り文字をスキップする

    // 次のバイトがコメントの始まり（#）であればコメント終了部分までスキップ，スキップしたならtrue，していなければfalseを返す
    function checkAndSkipComment(){
        if(contentBuffer[pos] === 35){// ASCII '#' = 35 (dec.)
            do{
                pos++;
            }
            while(contentBuffer[pos] !== 10 && contentBuffer[pos] !== 13);
            return true;
        }
        return false;
    }

    // 次のバイトから次の区切り文字手前までのASCII文字列を返す
    function getNextStrBeforeBlank(){
        tmp = [];
        while( pos < contentBuffer.length && !checkBlank(contentBuffer[pos]) ){
            tmp.push( contentBuffer[pos++] );
        }
        
        return String.fromCharCode(...tmp);
    }

    function addRect(x, y, r, g, b) {
        canvasCtx.fillStyle = `rgb(${r} ${g} ${b})`;
        canvasCtx.fillRect(x, y, 1, 1);
    }
    
    function appendCanvas(){
        isCanvasHidden = false;
        dropAreaObj.appendChild(canvasObj);
    }

    function drawPbmAscii() {
        // まだ下にコメントがあるかもしれないので処理
        skipBlank();// 区切り文字スキップ
        while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ

        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                if(pos >= contentBuffer.length){
                    message = "ラスターデータが欠けています";
                    return;
                }
                tmp = contentBuffer[pos++] === 48 ? 255 : 0;
                addRect(x, y, tmp, tmp, tmp);
                skipBlank();
            }
        }
        console.log(contentBuffer[pos-2],contentBuffer[pos-1],contentBuffer[pos], contentBuffer[pos+1],contentBuffer[pos+2]);
        appendCanvas();
    }

    function drawPgmAscii() {
        // まだ下にコメントがあるかもしれないので処理
        skipBlank();// 区切り文字スキップ
        while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ
        
        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp = getNextStrBeforeBlank();
                if(tmp === ""){
                    message = "ラスターデータが欠けています";
                    return;
                }
                tmp = parseInt(tmp) * brightnessScale;
                addRect(x, y, tmp, tmp, tmp);
                skipBlank();
            }
        }
        console.log(contentBuffer[pos-2],contentBuffer[pos-1],contentBuffer[pos], contentBuffer[pos+1],contentBuffer[pos+2]);
        appendCanvas();
    }

    function drawPpmAscii() {
        // まだ下にコメントがあるかもしれないので処理
        skipBlank();// 区切り文字スキップ
        while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ

        let tmp1;
        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp1 = [getNextStrBeforeBlank(), null, null];
                skipBlank();
                tmp1[1] = getNextStrBeforeBlank();
                skipBlank();
                tmp1[2] = getNextStrBeforeBlank();
                
                if(tmp1.indexOf("") >= 0){
                    message = "ラスターデータが欠けています";
                    return;
                }
                
                addRect(x, y, parseInt(tmp1[0]) * brightnessScale, parseInt(tmp1[1]) * brightnessScale, parseInt(tmp1[2]) * brightnessScale);
                skipBlank();
            }
        }
        console.log(contentBuffer[pos-2],contentBuffer[pos-1],contentBuffer[pos], contentBuffer[pos+1],contentBuffer[pos+2]);
        appendCanvas();
    }

    function drawPbmBin(){
        let tmp1;
        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ) {
                tmp = ("0000000" + contentBuffer[pos++].toString(2)).slice(-8);
                
                for(let i = 0; i < 8; i++){
                    if(x >= width) break;
                    tmp1 = tmp[i] === '0' ? 255 : 0;
                    addRect(x, y, tmp1, tmp1, tmp1);
                    ++x;
                }
            }
        }
        
        appendCanvas();
    }

    function drawPgmBin(){
        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp = contentBuffer[pos++];
                if(bytePerPixel === 2) tmp = tmp * 256 + contentBuffer[pos++];
                tmp *= brightnessScale;
                addRect(x, y, tmp, tmp, tmp);
            }
        }
        
        appendCanvas();
    }

    function drawPpmBin(){
        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp = [0, 0, 0];

                for(let i = 0; i < 3; i++){
                    tmp[i] = contentBuffer[pos++];
                    if(bytePerPixel === 2) tmp[i] = tmp[i] * 256 + contentBuffer[pos++];
                    tmp[i] *= brightnessScale;
                }
   
                addRect(x, y, tmp[0], tmp[1], tmp[2]);
            }
        }
        
        appendCanvas();
    }

    function reset(){
        if(canvasObj) canvasObj.remove(); // 元々あったcanvasを削除
        contentStr = "";
        fileName = fileSize = magicNum = width = height = maxBrightness = null;
        message = "処理中...";
        isCanvasHidden = true;
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

        reset();// ファイル情報変数をリセット

        file = event.dataTransfer.files[0];
        if (file) {
            if(!Object.values(pnmExt).includes( file.name.slice(-3) )){
                message = "PNMファイルのみ対応です。";
                return;
            }

            fileName = file.name;
            fileSize = (file?.size / 1024).toFixed(2);

            // PNMファイルを読み込む
            const reader = new FileReader();
            reader.onload = (e) => {
                // @ts-ignore
                contentBuffer = new Uint8Array(e.target.result);
                contentBuffer.forEach(a=>{
                    contentStr += a === 10 ? "<br>" : String.fromCharCode(a) ;// 10進数の10はASCII文字'\n'に相当
                });

                // ヘッダーを読み取る
                pos = 0;
                magicNum = getNextStrBeforeBlank();// マジックナンバー
                if(Object.values(magicNumbers).indexOf(magicNum) < 0){
                    message = "マジックナンバーが不正です。";
                    return;
                }
                skipBlank();// 区切り文字スキップ
                while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ
                width = parseInt( getNextStrBeforeBlank() );// 幅
                skipBlank();// 区切り文字スキップ
                while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ
                height = parseInt( getNextStrBeforeBlank() );// 高さ
                if (magicNum !== magicNumbers.PBM_A && magicNum !== magicNumbers.PBM_B){
                    skipBlank();// 区切り文字スキップ
                    while(checkAndSkipComment()) skipBlank();// 次がコメントならコメントとその後の区切り文字をスキップ
                    maxBrightness = parseInt( getNextStrBeforeBlank() );// ラスターデータ中の最大値
                    if(maxBrightness <= 0 || maxBrightness >= 65536){
                        message = "ラスターデータ中の最大値が不正です。";
                        return;
                    }
                    brightnessScale = 255 / maxBrightness;
                    bytePerPixel = maxBrightness < 256 ? 1 : 2;
                }
                pos++;// 区切り文字は1つのみ
                // P1~P3：まだ下にコメントがあるかもしれない → 別関数で処理
                // P4~P6：以降，posはラスターデータである


                // 画像表示用canvasを作成
                canvasObj = document.createElement("canvas");
                canvasObj.setAttribute("width", width.toString());
                canvasObj.setAttribute("height", height.toString());
                canvasCtx = canvasObj.getContext("2d");

                // 画像データ本体の処理
                if (magicNum === magicNumbers.PBM_A) drawPbmAscii();
                else if (magicNum === magicNumbers.PGM_A) drawPgmAscii();
                else if (magicNum === magicNumbers.PPM_A) drawPpmAscii();
                else if (magicNum === magicNumbers.PBM_B) drawPbmBin();
                else if (magicNum === magicNumbers.PGM_B) drawPgmBin();
                else if (magicNum === magicNumbers.PPM_B) drawPpmBin();
            };
            reader.readAsArrayBuffer(file);
        } else {
            message = "ファイルをアップロードしてください。";
        }
    }
</script>

<body>
    <p class="img-info">
        {magicNum || ""}
        {width || ""}
        {height || ""}
        { (magicNum !== magicNumbers.PBM_A && magicNum !== magicNumbers.PBM_B) ? maxBrightness || "" : ""}
    </p>

    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
        bind:this={dropAreaObj}
        id="drop-area"
        on:dragover={handleDragOver}
        on:dragleave={handleDragLeave}
        on:drop={handleDrop}
        class:hover={isHover}
    >
        {#if isCanvasHidden}
            {message}
        {/if}
    </div>

    <div class="file-info">
        ファイル名：{fileName || "---"}，サイズ：{fileSize || "---"} KB
        <br>
        【ファイルの中身】
        <div style="padding-left: 20px">
            {#if contentStr}
                {#if isContentOpen}
                    <button
                        on:click={() => {
                            isContentOpen = false;
                        }}>閉じる（軽くなるかも？）</button
                    >
                    <p
                        style="font-family: 'Courier New', Consolas, monospace, Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;"
                    >
                        {@html contentStr}
                    </p>
                {:else}
                    <button
                        on:click={() => {
                            isContentOpen = true;
                        }}>中身を表示</button
                    >
                {/if}
            {/if}
        </div>
    </div>
</body>

<style>
    #drop-area {
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
