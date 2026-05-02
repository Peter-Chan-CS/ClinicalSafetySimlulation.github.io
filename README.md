<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Clinical Safety Simulation</title>
    <style>
        :root {
            --bg-color: #f0f2f5;
            --npc-bubble: #ffffff;
            --player-bubble: #0084ff;
            --player-text: #ffffff;
            --text-main: #1c1e21;
            --accent: #42b72a;
            --danger: #fa3e3e;
            --shadow: 0 2px 4px rgba(0,0,0,0.1);
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg-color);
            margin: 0;
            display: flex;
            flex-direction: column;
            height: 100vh;
            overflow: hidden;
        }

        header {
            background: #fff;
            padding: 10px 15px;
            box-shadow: var(--shadow);
            display: flex;
            flex-direction: column;
            gap: 10px;
            z-index: 10;
        }

        .header-top { display: flex; justify-content: space-between; align-items: center; }
        .title { font-weight: 800; font-size: 1rem; color: #050505; }
        
        .header-actions {
            display: flex;
            gap: 8px;
            align-items: center;
        }

        .controls-row { display: flex; justify-content: space-between; align-items: center; }
        .score-box { font-weight: bold; color: var(--player-bubble); background: #e7f3ff; padding: 4px 12px; border-radius: 20px; font-size: 0.85rem; }
        
        #lang-toggle, .btn-header-reset { 
            padding: 5px 10px; 
            border-radius: 6px; 
            border: 1px solid #ddd; 
            background: #fff; 
            cursor: pointer; 
            font-size: 0.75rem;
            font-weight: 600;
            transition: all 0.2s;
        }

        .btn-header-reset {
            color: var(--danger);
            border-color: #ffcccc;
        }

        .btn-header-reset:hover { background: #fff0f0; border-color: var(--danger); }

        #chat-container {
            flex: 1;
            overflow-y: auto;
            padding: 20px;
            display: flex;
            flex-direction: column;
            gap: 15px;
            scroll-behavior: smooth;
        }

        .bubble {
            max-width: 85%;
            padding: 12px 16px;
            border-radius: 18px;
            font-size: 14px;
            line-height: 1.4;
            animation: fadeIn 0.3s ease forwards;
        }

        .bubble img {
            width: 100%;
            border-radius: 10px;
            margin-bottom: 8px;
            display: block;
        }

        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }

        .npc { align-self: flex-start; background-color: var(--npc-bubble); box-shadow: var(--shadow); border-bottom-left-radius: 4px; }
        .player { align-self: flex-end; background-color: var(--player-bubble); color: var(--player-text); border-bottom-right-radius: 4px; }
        .feedback { align-self: center; background: #fff3cd; color: #856404; font-size: 12px; font-style: italic; border-radius: 8px; padding: 10px; text-align: center; border: 1px solid #ffeeba; }

        #action-area {
            background: #fff;
            padding: 15px;
            border-top: 1px solid #ddd;
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .btn {
            display: flex;
            align-items: center;
            gap: 12px;
            padding: 10px;
            border: 1px solid #e4e6eb;
            border-radius: 12px;
            cursor: pointer;
            font-size: 13px;
            text-align: left;
            background: #f0f2f5;
            transition: background 0.2s;
        }

        .btn:hover { background: #e4e6eb; }
        .btn img { width: 50px; height: 50px; border-radius: 6px; object-fit: cover; flex-shrink: 0; }
        .btn-text { flex: 1; }

        .btn-primary { background: var(--player-bubble); color: white; justify-content: center; font-weight: bold; border: none; }

        #overlay {
            display: none;
            position: fixed;
            top:0; left:0; width:100%; height:100%;
            background: rgba(255,255,255,0.96);
            flex-direction: column;
            justify-content: center;
            align-items: center;
            z-index: 100;
        }

        .hidden { display: none !important; }
    </style>
</head>
<body>

<header>
    <div class="header-top">
        <div class="title" id="ui-title">Clinical Safety Simulation</div>
        <div class="header-actions">
            <button class="btn-header-reset" onclick="confirmReset()" id="ui-refresh-btn">↺ Reset</button>
            <select id="lang-toggle" onchange="changeLang(this.value)">
                <option value="en">English</option>
                <option value="zh">繁體中文</option>
            </select>
        </div>
    </div>
    <div class="controls-row">
        <div class="score-box"><span id="ui-score-label">Score</span>: <span id="score">0</span> / <span id="total-q">8</span></div>
    </div>
</header>

<div id="chat-container"></div>

<div id="action-area">
    <button id="optA" class="btn" onclick="handleChoice('A')">
        <img id="imgA" src="" class="hidden">
        <span id="textA" class="btn-text"></span>
    </button>
    <button id="optB" class="btn" onclick="handleChoice('B')">
        <img id="imgB" src="" class="hidden">
        <span id="textB" class="btn-text"></span>
    </button>
    <button id="restart" class="btn btn-primary hidden" onclick="resetGame()"></button>
</div>

<div id="overlay">
    <h1 id="ui-win-msg" style="color: var(--accent); font-size: 2.5rem; margin-bottom: 10px;">SUCCESS!</h1>
    <p id="ui-win-sub" style="margin-bottom: 20px;">Communication Skills Mastered.</p>
    <button class="btn btn-primary" style="width: 150px; padding: 15px;" onclick="resetGame()" id="ui-play-again">Restart</button>
</div>

<script>
    let currentLang = 'en';
    let currentQ = 0;
    let score = 0;

    const translations = {
        en: {
            title: "Clinical Safety Simulation",
            score: "Score",
            refresh: "↺ Reset",
            refreshConfirm: "Reset simulation? Your progress will be lost.",
            win: "SUCCESS!",
            winSub: "Communication Skills Mastered.",
            playAgain: "Restart Game",
            restart: "Restart Simulation",
            correct: "✅ Effective Communication.",
            incorrect: "❌ Risk of Escalation.",
            final: "Simulation complete. You scored "
        },
        zh: {
            title: "臨床安全模擬訓練",
            score: "得分",
            refresh: "↺ 重設",
            refreshConfirm: "確定重設模擬？目前的進度將會遺失。",
            win: "訓練達成！",
            winSub: "已掌握言語溝通與降溫技巧。",
            playAgain: "重新開始",
            restart: "重新開始模擬",
            correct: "✅ 有效溝通。",
            incorrect: "❌ 可能導致局勢升級。",
            final: "模擬結束。您的得分為 "
        }
    };

    const questions = [
        {
            images: {
                q: "https://images.unsplash.com/photo-1557050543-4d5f4e07ef46?w=400",
                a: "https://images.unsplash.com/photo-1540573133985-87b6da6d54a9?w=200",
                b: "https://images.unsplash.com/photo-1508061461508-cb18c242f556?w=200"
            },
            en: {
                q: "A patient is visibly upset about their breakfast being cold. How do you start the conversation?",
                a: "Use an open-ended question: 'I can see you're upset. Can you tell me more about what's happened?'",
                b: "Use a closed question: 'Is your food cold? I will go get another one for you now.'",
                feedback: "Open-ended questions allow the patient to vent and feel heard."
            },
            zh: {
                q: "患者對早餐變冷感到明顯不滿。您會如何開始對話？",
                a: "使用開放式問題：「我看得出您很不開心。能告訴我發生了什麼事嗎？」",
                b: "使用封閉式問題：「您的食物冷了嗎？我現在去幫您換一份。」",
                feedback: "開放式問題讓患者有機會宣洩情緒並感到被重視。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "The patient is speaking very loudly and quickly. How should you respond?",
                a: "Speak with a moderate speed and a firm, polite tone.",
                b: "Match their volume so they can hear you clearly.",
                feedback: "A moderate speed models calm behavior for the patient to mirror."
            },
            zh: {
                q: "患者說話聲音很大且速度很快。您應如何回應？",
                a: "以中等語速及堅定且有禮的語氣回覆。",
                b: "配合對方的音量說話，確保他們聽得見您的聲音。",
                feedback: "中等語速能為患者提供冷靜的榜樣，引導其模仿。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "You need the patient to sit down for a checkup.",
                a: "Give a simple and positive command: 'Please sit in this chair.'",
                b: "Use critical language: 'You shouldn't be standing right now.'",
                feedback: "Positive commands focus on what TO do rather than what NOT to do."
            },
            zh: {
                q: "您需要患者坐下進行檢查。",
                a: "給予簡單且正面的指令：「請坐在這張椅子上。」",
                b: "使用批評性語言：「您現在不應該站著。」",
                feedback: "正面的指令專注於「應該做什麼」，而非「不准做什麼」。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "The patient is confused about their meds. What do you do?",
                a: "Repeat important information clearly.",
                b: "Explain everything once very quickly to save time.",
                feedback: "Repetition confirms understanding and lowers anxiety."
            },
            zh: {
                q: "患者對用藥感到困惑。您會怎麼做？",
                a: "清晰地重複重要信息。",
                b: "快速解釋一遍所有內容以節省時間。",
                feedback: "重複重要信息能確認患者理解程度並降低焦慮。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "You are reviewing a patient's history of violence. What is your next move?",
                a: "Acknowledge the history and ensure security or a peer is nearby.",
                b: "Trust that they have changed and enter the area alone.",
                feedback: "Safety protocols exist to protect both staff and patients."
            },
            zh: {
                q: "您正在查閱患者的暴力病史。您的下一步是什麼？",
                a: "確認紀錄，並確保保安人員或同事在附近。",
                b: "相信他們已經改變，並獨自進入區域。",
                feedback: "安全規程的建立是為了同時保護工作人員與患者。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "A patient starts shouting profanities at you.",
                a: "Mind your tone: 'I want to help, but I cannot talk while you shout.'",
                b: "Defend yourself: 'You can't talk to me like that, it's rude.'",
                feedback: "Focusing on the task and boundaries is better than personal defense."
            },
            zh: {
                q: "患者開始對您辱罵。",
                a: "留意語氣：「我想幫助您，但大聲叫喊時我無法溝通。」",
                b: "為自己辯護：「你不能這樣跟我說話，這很不禮貌。」",
                feedback: "專注於任務與界限，比個人化的辯護更有效。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "The patient is in pain but acting aggressively.",
                a: "Offer appropriate comfort: 'I'm sorry you're in pain. Let's find help.'",
                b: "Distance yourself and ignore them until they are quiet.",
                feedback: "Kindness (no hostility) is a powerful de-escalation tool."
            },
            zh: {
                q: "患者抱怨疼痛但表現出攻擊性。",
                a: "提供適當的安慰：「抱歉您感到疼痛。我們來看看如何幫您。」",
                b: "保持距離，直到他們安靜為止都不予理會。",
                feedback: "友善（無敵意）是強大的降溫工具。"
            },
            correct: 'A'
        },
        {
            images: { q: "", a: "", b: "" },
            en: {
                q: "How do you confirm a patient understands the care plan?",
                a: "Open-ended: 'Can you tell me your understanding of this plan?'",
                b: "Closed: 'Is that clear to you?'",
                feedback: "Open-ended questions reveal actual comprehension gaps."
            },
            zh: {
                q: "您如何確認患者明白護理計劃？",
                a: "開放式：「您能告訴我您對這個計劃的理解嗎？」",
                b: "封閉式：「這對您來說清楚嗎？」",
                feedback: "開放式問題能揭示患者在理解上的實際差距。"
            },
            correct: 'A'
        }
    ];

    function updateUI() {
        const t = translations[currentLang];
        document.getElementById('ui-title').textContent = t.title;
        document.getElementById('ui-score-label').textContent = t.score;
        document.getElementById('ui-refresh-btn').textContent = t.refresh;
        document.getElementById('ui-win-msg').textContent = t.win;
        document.getElementById('ui-win-sub').textContent = t.winSub;
        document.getElementById('ui-play-again').textContent = t.playAgain;
        document.getElementById('restart').textContent = t.restart;
        document.getElementById('total-q').textContent = questions.length;
        
        if (currentQ < questions.length) {
            const q = questions[currentQ];
            document.getElementById('textA').textContent = q[currentLang].a;
            document.getElementById('textB').textContent = q[currentLang].b;
            handleBtnImage('imgA', q.images.a);
            handleBtnImage('imgB', q.images.b);
        }
    }

    function handleBtnImage(id, src) {
        const img = document.getElementById(id);
        if (src) {
            img.src = src;
            img.classList.remove('hidden');
        } else {
            img.classList.add('hidden');
        }
    }

    function changeLang(val) {
        currentLang = val;
        updateUI();
        const chat = document.getElementById('chat-container');
        if (chat.children.length > 0 && currentQ < questions.length) {
            const lastNpcMsg = Array.from(chat.children).reverse().find(m => m.classList.contains('npc') && !m.classList.contains('feedback'));
            if (lastNpcMsg && !lastNpcMsg.textContent.includes('✅') && !lastNpcMsg.textContent.includes('❌')) {
                const imgTag = lastNpcMsg.querySelector('img');
                lastNpcMsg.innerHTML = '';
                if (imgTag) lastNpcMsg.appendChild(imgTag);
                const textSpan = document.createElement('span');
                textSpan.textContent = questions[currentQ][currentLang].q;
                lastNpcMsg.appendChild(textSpan);
            }
        }
    }

    function confirmReset() {
        const t = translations[currentLang];
        if (confirm(t.refreshConfirm)) {
            resetGame();
        }
    }

    function init() {
        updateUI();
        if (currentQ < questions.length) {
            addMessage(questions[currentQ][currentLang].q, 'npc', questions[currentQ].images.q);
            document.getElementById('optA').classList.remove('hidden');
            document.getElementById('optB').classList.remove('hidden');
            document.getElementById('restart').classList.add('hidden');
        } else {
            endGame();
        }
    }

    function addMessage(text, type, imgSrc = null) {
        const div = document.createElement('div');
        div.className = `bubble ${type}`;
        if (imgSrc) {
            const img = document.createElement('img');
            img.src = imgSrc;
            div.appendChild(img);
        }
        const span = document.createElement('span');
        span.textContent = text;
        div.appendChild(span);
        const chat = document.getElementById('chat-container');
        chat.appendChild(div);
        chat.scrollTop = chat.scrollHeight;
    }

    function handleChoice(choice) {
        const qData = questions[currentQ];
        const t = translations[currentLang];
        const choiceLower = choice.toLowerCase();
        
        addMessage(qData[currentLang][choiceLower], 'player', qData.images[choiceLower]);
        document.getElementById('optA').classList.add('hidden');
        document.getElementById('optB').classList.add('hidden');

        setTimeout(() => {
            if (choice === qData.correct) {
                score++;
                document.getElementById('score').textContent = score;
                addMessage(t.correct, 'npc');
            } else {
                addMessage(t.incorrect, 'npc');
            }
            addMessage(qData[currentLang].feedback, 'feedback');
            currentQ++;

            setTimeout(() => {
                if(score >= 7 && currentQ === questions.length) {
                    document.getElementById('overlay').style.display = 'flex';
                } else if (currentQ < questions.length) {
                    init();
                } else {
                    endGame();
                }
            }, 1200);
        }, 600);
    }

    function endGame() {
        const t = translations[currentLang];
        addMessage(`${t.final} ${score} / ${questions.length}.`, 'npc');
        document.getElementById('restart').classList.remove('hidden');
    }

    function resetGame() {
        score = 0;
        currentQ = 0;
        document.getElementById('score').textContent = "0";
        document.getElementById('chat-container').innerHTML = '';
        document.getElementById('overlay').style.display = 'none';
        init();
    }

    init();
</script>
</body>
</html>
