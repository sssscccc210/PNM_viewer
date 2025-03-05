<script>
    const pnmExt = {
        PBM: "pbm",
        PGM: "pgm",
        PPM: "ppm",
    };
    const magicNumbers = {
        PBM: "P1",
        PGM: "P2",
        PPM: "P3",
    };
    let isHover = false;
    let isContentOpen = false;
    let file;
    let fileName = "---";
    let fileSize = "---";
    let content; // 中身そのまま表示用html
    let dropAreaObj; // D&D area obj
    let canvasObj; // canvasObj
    let canvasCtx; // context of canvasObj
    let isCanvasHidden = true;
    let message = "ここにファイルをドラッグ＆ドロップ";
    let brightnessScale; // maxBrightnessを基準にした255に対する倍率
    let magicNum, width, height, maxBrightness;

    let tmp;

    function addRect(x, y, r, g, b) {
        canvasCtx.fillStyle = `rgb(${r} ${g} ${b})`;
        canvasCtx.fillRect(x, y, 1, 1);
    }
    
    function appendCanvas(){
        isCanvasHidden = false;
        canvasCtx.scale(10, 10);
        dropAreaObj.appendChild(canvasObj);
    }

    function drawPbm(d) {
        let data = [];
        let index = 0;

        // PBM形式は"011001"のように空白文字がないパターンもある
        for (let i = 3; i < d.length; ++i){
            data.push.apply(data, d[i].split('')); // 参考：https://qiita.com/jkr_2255/items/11e80010602d3387533b
        }

        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                /*
                * viteの不具合(?)が原因で，
                * if(!data[index]){ ... return; }
                * だといきなりif内が実行されてreturnするため，
                * 一回tmpに代入してからifで評価するとうまくいく
                */
                tmp = !data[index];
                if(tmp){
                    message = "ラスターデータが欠けています";
                    return;
                }
                addRect(x, y, (data[index] - 1) * -255, (data[index] - 1) * -255, (data[index] - 1) * -255);
                index++;
            }
        }

        appendCanvas();
    }

    function drawPgm(data) {
        let index = 4;

        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp = !data[index];
                if(tmp){
                    message = "ラスターデータが欠けています";
                    return;
                }
                addRect(x, y, data[index] * brightnessScale, data[index] * brightnessScale, data[index] * brightnessScale);
                index++;
            }
        }

        appendCanvas();
    }

    function drawPpm(data) {
        let index = 4;

        for (let y = 0; y < height; ++y) {
            for (let x = 0; x < width; ++x) {
                tmp = !data[index];
                if(tmp){
                    message = "ラスターデータが欠けています";
                    return;
                }
                addRect(x, y, data[index] * brightnessScale, data[index + 1] * brightnessScale, data[index + 2] * brightnessScale);
                index += 3;
            }
        }

        appendCanvas();
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

        // 元々あったcanvasを削除
        if(canvasObj) canvasObj.remove();

        message = "処理中...";
        isCanvasHidden = true;

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
            reader.readAsText(file);
            reader.onload = (e) => {
                let noCommentData = e.target.result
                    .toString() // 読み込んだデータをstringに変換
                    .replaceAll(/\#.*?(?=\r|\n|$)/g, '') // コメント部を削除（#からCRまたはLFまたはEOFの直前まで）
                    .split(/\s/) // 区切り文字を基にsplit
                    .filter((a) => a != ""); // 空要素を削除
                content = e.target.result.toString().replaceAll("\n", "<br>"); // 中身そのまま表示用
                //console.log(noCommentData)
                
                // 画像情報（ヘッダー）を格納
                magicNum = noCommentData[0];
                width = parseInt(noCommentData[1]);
                height = parseInt(noCommentData[2]);
                if (magicNum !== magicNumbers.PBM){
                    maxBrightness = parseInt(noCommentData[3]);
                    brightnessScale = 255 / maxBrightness;
                }

                // 画像表示用canvasを作成
                canvasObj = document.createElement("canvas");
                canvasObj.setAttribute("width", width.toString());
                canvasObj.setAttribute("height", height.toString());
                canvasCtx = canvasObj.getContext("2d");

                // 画像データ本体の処理
                if (magicNum === magicNumbers.PBM) drawPbm(noCommentData);
                else if (magicNum === magicNumbers.PGM) drawPgm(noCommentData);
                else if (magicNum === magicNumbers.PPM) drawPpm(noCommentData);
                else message = "マジックナンバーが不正です。";
            };
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
        {magicNum !== magicNumbers.PBM ? maxBrightness || "" : ""}
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
        ファイル名：{fileName}，サイズ：{fileSize} KB
        <br>
        【ファイルの中身】
        <div style="padding-left: 20px">
            {#if content}
                {#if isContentOpen}
                    <button
                        on:click={() => {
                            isContentOpen = false;
                        }}>閉じる（軽くなるかも？）</button
                    >
                    <p
                        style="font-family: 'Courier New', Consolas, monospace, Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;"
                    >
                        {@html content}
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
