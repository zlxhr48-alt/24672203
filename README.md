<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 포즈 따라하기 게임</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.4.0/p5.js"></script>
    <script src="https://unpkg.com/ml5@latest/dist/ml5.min.js"></script>
    <style>
        :root {
            --primary: #4a90e2;
            --success: #4caf50;
            --fail: #f44336;
            --accent: #ffeb3b;
        }

        body { 
            background: linear-gradient(135deg, #e0f2fe 0%, #f0f9ff 100%);
            font-family: 'Pretendard', 'Nanum Gothic', sans-serif; 
            display: flex; 
            flex-direction: column; 
            align-items: center; 
            justify-content: center; 
            min-height: 100vh; 
            margin: 0; 
            overflow: hidden; 
        }

        #start-screen, #result-screen {
            text-align: center;
            background: white;
            padding: 40px;
            border-radius: 30px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.1);
            z-index: 20;
            max-width: 450px;
            width: 90%;
        }

        #result-screen { display: none; }

        h1 { color: var(--primary); margin-bottom: 10px; font-size: 2.2rem; }
        .score-display { font-size: 4rem; font-weight: bold; color: var(--primary); margin: 20px 0; }
        p { color: #666; margin-bottom: 20px; line-height: 1.5; }
        
        .instruction-box {
            background: #f8f9fa;
            border-left: 5px solid var(--primary);
            padding: 15px;
            margin-bottom: 25px;
            text-align: left;
            font-size: 0.95rem;
            color: #444;
        }

        /* 캔버스 컨테이너 중앙 정렬 강화 */
        #canvas-container { 
            position: relative; 
            border: 8px solid white; 
            border-radius: 24px; 
            overflow: hidden; 
            box-shadow: 0 20px 40px rgba(0,0,0,0.15); 
            display: none;
            line-height: 0;
        }

        canvas {
            display: block;
            margin: 0 auto;
        }

        .info-panel { 
            position: absolute; 
            top: 0; 
            width: 100%; 
            padding: 20px 0;
            text-align: center; 
            background: linear-gradient(to bottom, rgba(0,0,0,0.7), transparent);
            color: white; 
            pointer-events: none; 
            z-index: 10;
        }

        .mission-label { font-size: 1.2rem; opacity: 0.9; margin-bottom: 5px; }
        .mission-target { font-size: 3.2rem; font-weight: 800; color: var(--accent); text-shadow: 0 4px 8px rgba(0,0,0,0.3); }

        .guide-panel {
            position: absolute;
            bottom: 80px;
            width: 100%;
            text-align: center;
            pointer-events: none;
            z-index: 10;
        }

        #guide-text {
            background: rgba(0, 0, 0, 0.6);
            color: white;
            padding: 12px 25px;
            border-radius: 30px;
            display: inline-block;
            font-size: 1.1rem;
            border: 2px solid rgba(255, 255, 255, 0.3);
            backdrop-filter: blur(5px);
            max-width: 85%;
            word-break: keep-all;
        }

        .status-container {
            position: absolute;
            top: 20px;
            right: 20px;
            display: flex;
            flex-direction: column;
            gap: 10px;
            align-items: flex-end;
            z-index: 10;
        }

        .status-item {
            background: rgba(0,0,0,0.5);
            padding: 8px 18px;
            border-radius: 50px;
            color: white;
            font-size: 1.3rem;
            font-weight: bold;
        }

        .score-board { 
            position: absolute; 
            bottom: 20px; 
            left: 50%; 
            transform: translateX(-50%);
            background: white;
            padding: 12px 35px;
            border-radius: 50px;
            box-shadow: 0 5px 15px rgba(0,0,0,0.1);
            font-size: 1.6rem;
            font-weight: bold;
            color: var(--primary);
            z-index: 10;
        }

        button { 
            padding: 16px 36px; 
            font-size: 1.3rem; 
            font-weight: bold;
            background-color: var(--primary); 
            color: white; 
            border: none; 
            border-radius: 15px; 
            cursor: pointer; 
            transition: all 0.2s; 
            box-shadow: 0 5px 0 #2a6dbd;
        }

        button:active { transform: translateY(3px); box-shadow: 0 2px 0 #2a6dbd; }

        #feedback-overlay {
            position: absolute;
            top: 0; left: 0; width: 100%; height: 100%;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 6rem;
            pointer-events: none;
            opacity: 0;
            transition: opacity 0.2s;
            z-index: 15;
        }

        .flash-success { background: rgba(76, 175, 80, 0.3); animation: flash 0.6s; }
        .flash-fail { background: rgba(244, 67, 54, 0.3); animation: flash 0.6s; }

        @keyframes flash {
            0% { opacity: 0; }
            50% { opacity: 1; }
            100% { opacity: 0; }
        }
    </style>
</head>
<body>

    <!-- 시작 화면 -->
    <div id="start-screen">
        <h1>🕺 AI 포즈 챌린지 💃</h1>
        <p>무작위로 나오는 3개의 포즈 미션을 수행하세요!</p>
        
        <div class="instruction-box">
            <strong>📢 게임 방법:</strong><br>
            1. 카메라 앞에서 <strong>일어서서</strong> 전신이 잘 보이게 준비하세요.<br>
            2. 화면에 표시되는 포즈를 5초 안에 정확히 따라하세요.<br>
            3. AI가 여러분의 동작을 인식하여 점수를 계산합니다.
        </div>

        <div id="loading-msg" style="margin-bottom: 20px; color: #888;">AI 모델을 불러오는 중입니다...</div>
        <button id="start-btn" onclick="startGame()" style="display:none;">게임 시작하기</button>
    </div>

    <!-- 결과 화면 -->
    <div id="result-screen">
        <h1>🎮 게임 종료!</h1>
        <p>당신의 최종 점수는?</p>
        <div id="final-score" class="score-display">0</div>
        <button onclick="restartGame()">다시 도전하기</button>
    </div>

    <!-- 게임 화면 -->
    <div id="canvas-container">
        <div id="feedback-overlay"></div>
        
        <div class="info-panel">
            <div id="round-text" class="mission-label">미션 1 / 3</div>
            <div id="mission-text" class="mission-target">준비...</div>
        </div>

        <div class="guide-panel">
            <div id="guide-text">포즈를 취해 보세요!</div>
        </div>

        <div class="status-container">
            <div class="status-item">⏱ <span id="timer-text">5</span>s</div>
        </div>

        <div id="score-text" class="score-board">SCORE: 000</div>
    </div>

    <script>
        let classifier;
        let video;
        let modelURL = 'https://teachablemachine.withgoogle.com/models/3D-S4mtMp/';
        
        const poseInfo = {
            "만세": "두 손바닥이 쫙 보이게 피고 만세!",
            "스쿼트": "정면을 보면서 스쿼트 자세를 취하세요.",
            "하트": "머리 위로 팔을 올려 큰 하트를 만드세요!",
            "궁수": "왼쪽 손을 앞으로 쭉 뻗으면서 활 쏘는 포즈!",
            "경례": "왼쪽 손으로 씩씩하게 경례하세요!"
        };

        const poses = Object.keys(poseInfo);
        let currentMission = "";
        let score = 0;
        let timeLeft = 5;
        let isGameRunning = false;
        let timerInterval;
        
        let missionCount = 0;
        const MAX_MISSIONS = 3;

        function preload() {
            classifier = ml5.imageClassifier(modelURL + 'model.json', () => {
                document.getElementById('loading-msg').innerText = "인식 준비 완료!";
                document.getElementById('start-btn').style.display = "inline-block";
            });
        }

        function setup() {
            const canvas = createCanvas(640, 480);
            canvas.parent('canvas-container');
            video = createCapture(VIDEO);
            video.size(640, 480);
            video.hide();
        }

        function startGame() {
            document.getElementById('start-screen').style.display = 'none';
            document.getElementById('result-screen').style.display = 'none';
            document.getElementById('canvas-container').style.display = 'inline-block';
            
            isGameRunning = true;
            score = 0;
            missionCount = 0;
            
            updateScore();
            nextMission();
            classifyVideo();
        }

        function restartGame() {
            startGame();
        }

        function classifyVideo() {
            if (!isGameRunning) return;
            classifier.classify(video, (error, results) => {
                if (error) return;
                
                let label = results[0].label;
                let confidence = results[0].confidence;

                if (label === currentMission && confidence > 0.85) {
                    handleSuccess();
                } else {
                    classifyVideo();
                }
            });
        }

        function nextMission() {
            if (!isGameRunning) return;

            missionCount++;
            if (missionCount > MAX_MISSIONS) {
                endGame();
                return;
            }

            document.getElementById('round-text').innerText = `미션 ${missionCount} / ${MAX_MISSIONS}`;

            let next;
            do {
                next = poses[Math.floor(Math.random() * poses.length)];
            } while (next === currentMission);

            currentMission = next;
            document.getElementById('mission-text').innerText = currentMission;
            document.getElementById('guide-text').innerText = `💡 ${poseInfo[currentMission]}`;
            
            timeLeft = 5;
            updateTimerUI();
            
            clearInterval(timerInterval);
            timerInterval = setInterval(() => {
                timeLeft--;
                updateTimerUI();
                if (timeLeft <= 0) {
                    handleFail();
                }
            }, 1000);
        }

        function handleSuccess() {
            if (!isGameRunning) return;
            clearInterval(timerInterval);
            
            score += 10;
            updateScore();
            
            showFeedback("✨ SUCCESS ✨", "flash-success");
            
            setTimeout(() => {
                nextMission();
                if (isGameRunning) classifyVideo();
            }, 1200);
        }

        function handleFail() {
            if (!isGameRunning) return;
            clearInterval(timerInterval);
            
            showFeedback("❌ TIME OUT ❌", "flash-fail");
            
            setTimeout(() => {
                nextMission();
                if (isGameRunning) classifyVideo();
            }, 1200);
        }

        function endGame() {
            isGameRunning = false;
            clearInterval(timerInterval);
            
            document.getElementById('canvas-container').style.display = 'none';
            document.getElementById('result-screen').style.display = 'block';
            document.getElementById('final-score').innerText = score;
        }

        function showFeedback(text, className) {
            const overlay = document.getElementById('feedback-overlay');
            overlay.innerText = text;
            overlay.className = className;
            overlay.style.opacity = "1";
            overlay.style.fontSize = "4rem";
            overlay.style.color = "white";
            overlay.style.fontWeight = "bold";
            overlay.style.textShadow = "0 0 20px rgba(0,0,0,0.5)";

            setTimeout(() => { overlay.style.opacity = "0"; }, 800);
        }

        function updateScore() {
            document.getElementById('score-text').innerText = `SCORE: ${String(score).padStart(3, '0')}`;
        }

        function updateTimerUI() {
            document.getElementById('timer-text').innerText = timeLeft;
            const timerSpan = document.getElementById('timer-text');
            if (timeLeft <= 2) {
                timerSpan.style.color = "#ff5252";
            } else {
                timerSpan.style.color = "white";
            }
        }

        function draw() {
            push();
            translate(width, 0);
            scale(-1, 1);
            image(video, 0, 0, width, height);
            pop();
        }
    </script>
</body>
</html>
