# my-app-alrmwacth-
HTML
<!DOCTYPE html><html lang="ja"><head>
    <meta charset="UTF-8">
    <title>タイムリミット・ホワイトクロック（音声機能付き）</title>
    <style>
        body {
            display: flex;
            flex-direction: column; /* ボタン配置のために縦並びに */
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #eef2f3;
            margin: 0;
            font-family: 'Helvetica Neue', Arial, sans-serif;
        }

        /* 🕰️ 文字盤 */
        .clock {
            width: 300px;
            height: 300px;
            background-color: #ffffff;
            border: 10px solid #333333;
            border-radius: 50%;
            position: relative;
            box-shadow: 0 8px 24px rgba(0, 0, 0, 0.1);
            overflow: hidden;
            margin-bottom: 20px;
        }

        /* 🔢 時字（数字） */
        .number {
            position: absolute;
            top: 50%;
            left: 50%;
            font-weight: bold;
            font-size: 22px;
            color: #333333;
            z-index: 5;
            text-shadow: 0 0 6px #ffffff, 0 0 3px #ffffff;
        }

        /* 📍 針の共通設定 */
        .hand {
            position: absolute;
            bottom: 50%;
            left: 50%;
            transform-origin: bottom center;
            border-radius: 4px;
            z-index: 10;
        }

        .hour { width: 8px; height: 70px; background-color: #333333; margin-left: -4px; }
        .minute { width: 5px; height: 110px; background-color: #555555; margin-left: -2.5px; }
        .second { width: 2px; height: 120px; background-color: #111111; margin-left: -1px; }

        .center-dot {
            width: 14px; height: 14px; background-color: #333333; border-radius: 50%;
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 20;
        }

        /* 🔘 スタートボタン */
        .start-btn {
            padding: 10px 24px;
            font-size: 16px;
            font-weight: bold;
            color: #ffffff;
            background-color: #4cab50; /* ずんだカラー */
            border: none;
            border-radius: 20px;
            cursor: pointer;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: background 0.2s;
        }
        .start-btn:hover { background-color: #439647; }
        .start-btn.playing { background-color: #e53935; } /* 停止時は赤っぽく */
    </style></head><body>

    <div class="clock" id="clock-face">
        <div class="hand hour" id="hour-hand"></div>
        <div class="hand minute" id="minute-hand"></div>
        <div class="hand second" id="second-hand"></div>
        <div class="center-dot"></div>
    </div>

    <button class="start-btn" id="audio-toggle-btn">サウンドをONにする（ノイズ開始）</button>

    <script>
        const clockFace = document.getElementById('clock-face');
        const hourHand = document.getElementById('hour-hand');
        const minuteHand = document.getElementById('minute-hand');
        const secondHand = document.getElementById('second-hand');
        const audioToggleBtn = document.getElementById('audio-toggle-btn');

        // 1. 【ずんだもん音声の設定】
        // ★それぞれのタイミングのGoogleドライブのIDに書き換えてください
        const audioFiles = {
            15: new Audio('https://docs.google.com/uc?export=download&id=12GF1buhFr9E0yhaBCqIXIS7DfbxBQIZ4'),
            30: new Audio('https://docs.google.com/uc?export=download&id=1RQL2j8brDx-oJV6rbNEsLPLWLSQPB9XL'),
            40: new Audio('https://docs.google.com/uc?export=download&id=164Tsnd7fy_qXd4nKUQdziQUuRHJhTehD'),
            50: new Audio('https://docs.google.com/uc?export=download&id=10Ymbnj2pCI61s1o9CB-yd-5u2m4ul3QC')
        };

        // 音声がその分で再生済みかを記録するフラグ
        let lastPlayedMinute = -1;

        // 2. 【ピンクノイズの生成システム（Web Audio API）】
        let audioCtx = null;
        let noiseNode = null;
        let gainNode = null;
        let isNoisePlaying = false;

        function createPinkNoise() {
            const bufferSize = 4 * audioCtx.sampleRate;
            const noiseBuffer = audioCtx.createBuffer(1, bufferSize, audioCtx.sampleRate);
            const output = noiseBuffer.getChannelData(0);
            
            let b0=0, b1=0, b2=0, b3=0, b4=0, b5=0, b6=0;
            for (let i = 0; i < bufferSize; i++) {
                let white = Math.random() * 2 - 1;
                b0 = 0.99886 * b0 + white * 0.0555179;
                b1 = 0.99332 * b1 + white * 0.0750759;
                b2 = 0.96900 * b2 + white * 0.1538520;
                b3 = 0.86650 * b3 + white * 0.3104856;
                b4 = 0.55000 * b4 + white * 0.5329522;
                b5 = -0.7616 * b5 - white * 0.0168980;
                output[i] = b0 + b1 + b2 + b3 + b4 + b5 + b6 + white * 0.5362;
                output[i] *= 0.11;
                b6 = white * 0.115926;
            }

            const node = audioCtx.createBufferSource();
            node.buffer = noiseBuffer;
            node.loop = true;
            return node;
        }

        function toggleAudio() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }

            if (!isNoisePlaying) {
                noiseNode = createPinkNoise();
                gainNode = audioCtx.createGain();
                gainNode.gain.value = 0.35; 

                noiseNode.connect(gainNode);
                gainNode.connect(audioCtx.destination);
                noiseNode.start();
                
                audioToggleBtn.innerText = "サウンドをOFFにする";
                audioToggleBtn.classList.add('playing');
                isNoisePlaying = true;
            } else {
                if (noiseNode) {
                    noiseNode.stop();
                    noiseNode.disconnect();
                }
                audioToggleBtn.innerText = "サウンドをONにする（ノイズ開始）";
                audioToggleBtn.classList.remove('playing');
                isNoisePlaying = false;
            }
        }

        audioToggleBtn.addEventListener('click', toggleAudio);

        // 3. 時字（1〜12）の自動生成
        for (let i = 1; i <= 12; i++) {
            const numDiv = document.createElement('div');
            numDiv.className = 'number';
            numDiv.innerText = i;
            const angle = i * 30; 
            numDiv.style.transform = `translate(-50%, -50%) rotate(${angle}deg) translateY(-110px) rotate(${-angle}deg)`;
            clockFace.appendChild(numDiv);
        }

        // 4. 時計の更新＆音声トリガー処理
        function updateClock() {
            const now = new Date();
            const hours = now.getHours();
            const minutes = now.getMinutes();
            const seconds = now.getSeconds();

            // 針の回転
            const hourDeg = (hours % 12) * 30 + minutes * 0.5;
            const minuteDeg = minutes * 6 + seconds * 0.1;
            const secondDeg = seconds * 6;

            hourHand.style.transform = `rotate(${hourDeg}deg)`;
            minuteHand.style.transform = `rotate(${minuteDeg}deg)`;
            secondHand.style.transform = `rotate(${secondDeg}deg)`;

            // 背景グラデーション
            const totalSeconds = minutes * 60 + seconds;
            const limitSeconds = 50 * 60; // 50分

            if (totalSeconds < limitSeconds) {
                const progress = totalSeconds / limitSeconds;
                const whiteEnd = 100 - (progress * 100);
                const orangeEnd = 115 - (progress * 115);
                const redEnd = 130 - (progress * 130);
                
                clockFace.style.background = `radial-gradient(circle, #ffffff ${whiteEnd}%, #ff9500 ${orangeEnd}%, #ff3b30 ${redEnd}%)`;
            } else {
                clockFace.style.background = '#ffffff';
            }

            // 💡【音声トリガーシステム】
            // 15分、30分、40分、50分のちょうど0秒のタイミングで判定
            if (seconds === 0 && lastPlayedMinute !== minutes) {
                if (audioFiles[minutes]) {
                    audioFiles[minutes].play().catch(error => {
                        console.log(`${minutes}分の音声再生がブロックされました。画面をクリックするかボタンをONにしてください:`, error);
                    });
                    lastPlayedMinute = minutes; // 連続で何回も再生されないようにロック
                }
            }

            // 秒が動いたらフラグのロックを外す準備（安全対策）
            if (seconds > 1) {
                lastPlayedMinute = -1;
            }
        }

        // 1秒ごとに実行
        setInterval(updateClock, 1000);
        updateClock();
    </script></body></html>
