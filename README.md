[index.html](https://github.com/user-attachments/files/24557720/index.html)
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 독서 퀴즈 메이커</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@300;400;500;700&display=swap');
        body { font-family: 'Noto Sans KR', sans-serif; }
        .glass { background: rgba(255, 255, 255, 0.95); backdrop-filter: blur(10px); }
        .loading-dots:after { content: '.'; animation: dots 1.5s steps(5, end) infinite; }
        @keyframes dots { 0%, 20% { content: '.'; } 40% { content: '..'; } 60% { content: '...'; } 80%, 100% { content: ''; } }
    </style>
</head>
<body class="bg-slate-50 min-h-screen text-slate-800">

    <div class="max-w-4xl mx-auto px-4 py-12">
        <!-- Header -->
        <header class="text-center mb-12">
            <h1 class="text-4xl font-bold text-indigo-700 mb-4">📚 AI 독서 퀴즈 메이커</h1>
            <p class="text-slate-600">책의 내용을 입력하면 AI가 학습용 퀴즈를 만들어 드립니다.</p>
        </header>

        <!-- Step 1: Input Section -->
        <section id="input-section" class="glass p-8 rounded-2xl shadow-xl border border-slate-200 mb-8">
            <div class="mb-6">
                <label class="block text-sm font-semibold text-slate-700 mb-2">책 내용 또는 요약 텍스트</label>
                <textarea id="book-content" rows="10" 
                    class="w-full p-4 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none transition-all"
                    placeholder="여기에 문제의 기반이 될 책의 본문이나 내용을 붙여넣어 주세요..."></textarea>
            </div>
            
            <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
                <div>
                    <label class="block text-sm font-semibold text-slate-700 mb-2">문제 개수</label>
                    <select id="quiz-count" class="w-full p-3 rounded-xl border border-slate-300 outline-none">
                        <option value="3">3문제</option>
                        <option value="5" selected>5문제</option>
                        <option value="10">10문제</option>
                    </select>
                </div>
                <div>
                    <label class="block text-sm font-semibold text-slate-700 mb-2">문제 유형</label>
                    <select id="quiz-type" class="w-full p-3 rounded-xl border border-slate-300 outline-none">
                        <option value="multiple">객관식</option>
                        <option value="mixed">객관식 + 주관식 혼합</option>
                    </select>
                </div>
            </div>

            <button onclick="generateQuiz()" id="generate-btn" 
                class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-4 rounded-xl transition-all shadow-lg shadow-indigo-200">
                퀴즈 생성하기
            </button>
        </section>

        <!-- Loading View -->
        <div id="loading" class="hidden text-center py-20">
            <div class="inline-block animate-spin rounded-full h-12 w-12 border-4 border-indigo-500 border-t-transparent mb-4"></div>
            <p class="text-xl font-medium text-slate-700">AI가 책 내용을 읽고 문제를 만들고 있습니다<span class="loading-dots"></span></p>
        </div>

        <!-- Step 2: Quiz View Section -->
        <section id="quiz-section" class="hidden space-y-6">
            <div id="quiz-container"></div>
            <div class="flex gap-4">
                <button onclick="resetApp()" class="flex-1 bg-slate-200 hover:bg-slate-300 text-slate-700 font-bold py-4 rounded-xl transition-all">
                    다시 하기
                </button>
                <button onclick="checkAnswers()" class="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-4 rounded-xl transition-all shadow-lg shadow-green-200">
                    정답 확인하기
                </button>
            </div>
        </section>

        <!-- Result View -->
        <div id="result-modal" class="hidden fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div class="bg-white rounded-2xl p-8 max-w-md w-full text-center shadow-2xl">
                <h2 class="text-3xl font-bold mb-2">결과 확인</h2>
                <div id="score-display" class="text-5xl font-bold text-indigo-600 my-6">0 / 0</div>
                <p id="result-message" class="text-slate-600 mb-8"></p>
                <button onclick="closeModal()" class="w-full bg-indigo-600 text-white font-bold py-3 rounded-xl">상세 풀이 보기</button>
            </div>
        </div>
    </div>

    <script>
        // API 키가 비어있는 경우를 대비한 체크 (Canvas 환경에서는 런타임에 주입되거나 수동 입력이 필요합니다)
        const apiKey = ""; 
        let currentQuizData = [];

        async function generateQuiz() {
            const content = document.getElementById('book-content').value;
            const count = document.getElementById('quiz-count').value;
            const type = document.getElementById('quiz-type').value;

            if (!content.trim()) {
                alert("책 내용을 입력해 주세요.");
                return;
            }

            // UI 전환
            document.getElementById('input-section').classList.add('hidden');
            document.getElementById('loading').classList.remove('hidden');

            const systemPrompt = `당신은 독서 교육 전문가입니다. 사용자가 입력한 텍스트를 바탕으로 독해 능력을 측정하는 퀴즈를 생성하세요. 
            반드시 다음 JSON 구조로 응답하세요:
            {
                "quizzes": [
                    {
                        "id": 1,
                        "question": "문제 내용",
                        "options": ["보기1", "보기2", "보기3", "보기4"],
                        "answer": 0,
                        "explanation": "해설 내용",
                        "type": "multiple"
                    }
                ]
            }
            - 문제 개수: ${count}개
            - 유형: ${type === 'mixed' ? '객관식과 주관식 혼합' : '객관식 위주'}`;

            try {
                const response = await fetchWithRetry(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: `내용: ${content}` }] }],
                        systemInstruction: { parts: [{ text: systemPrompt }] },
                        generationConfig: { responseMimeType: "application/json" }
                    })
                });

                if (!response.ok) {
                    const errorData = await response.json();
                    throw new Error(errorData.error?.message || "API 요청에 실패했습니다.");
                }

                const data = await response.json();
                const textResponse = data.candidates[0].content.parts[0].text;
                
                // JSON만 추출하기 위한 정규식 (마크다운 코드 블록 등 방지)
                const jsonMatch = textResponse.match(/\{[\s\S]*\}/);
                if (!jsonMatch) throw new Error("유효한 퀴즈 데이터를 받지 못했습니다.");
                
                const result = JSON.parse(jsonMatch[0]);
                currentQuizData = result.quizzes;
                renderQuiz();
            } catch (error) {
                console.error("Error generating quiz:", error);
                alert(`오류가 발생했습니다: ${error.message}\n\n도움말: API 키가 올바른지 확인해 보세요.`);
                resetApp();
            } finally {
                document.getElementById('loading').classList.add('hidden');
            }
        }

        async function fetchWithRetry(url, options, retries = 5) {
            for (let i = 0; i < retries; i++) {
                try {
                    const response = await fetch(url, options);
                    // 400번대 에러(인증 실패 등)는 재시도하지 않고 즉시 반환
                    if (response.status >= 400 && response.status < 500) return response;
                    if (response.ok) return response;
                } catch (e) {
                    if (i === retries - 1) throw e;
                }
                await new Promise(res => setTimeout(res, Math.pow(2, i) * 1000));
            }
            throw new Error("네트워크 연결 상태가 좋지 않습니다.");
        }

        function renderQuiz() {
            const container = document.getElementById('quiz-container');
            container.innerHTML = '';
            
            currentQuizData.forEach((quiz, index) => {
                const quizDiv = document.createElement('div');
                quizDiv.className = 'glass p-6 rounded-2xl shadow border border-slate-200';
                
                let contentHtml = `<h3 class="text-lg font-bold mb-4"><span class="text-indigo-600">Q${index+1}.</span> ${quiz.question}</h3>`;
                
                if (quiz.type === 'multiple' || (quiz.options && quiz.options.length > 0)) {
                    contentHtml += `<div class="space-y-2">`;
                    quiz.options.forEach((opt, optIdx) => {
                        contentHtml += `
                            <label class="flex items-center p-3 border border-slate-200 rounded-lg cursor-pointer hover:bg-indigo-50 transition-colors">
                                <input type="radio" name="quiz-${index}" value="${optIdx}" class="w-4 h-4 text-indigo-600 mr-3">
                                <span>${opt}</span>
                            </label>`;
                    });
                    contentHtml += `</div>`;
                } else {
                    contentHtml += `
                        <input type="text" name="quiz-${index}" placeholder="정답을 입력하세요" 
                            class="w-full p-3 border border-slate-300 rounded-lg outline-none focus:ring-2 focus:ring-indigo-500">`;
                }
                
                const displayAnswer = (quiz.type === 'multiple' || quiz.options) ? quiz.options[quiz.answer] : quiz.answer;
                
                contentHtml += `
                    <div id="explanation-${index}" class="hidden mt-4 p-4 bg-amber-50 rounded-lg border border-amber-200 text-sm">
                        <p class="font-bold text-amber-800 mb-1">정답: ${displayAnswer}</p>
                        <p class="text-slate-700">${quiz.explanation}</p>
                    </div>`;

                quizDiv.innerHTML = contentHtml;
                container.appendChild(quizDiv);
            });

            document.getElementById('quiz-section').classList.remove('hidden');
            window.scrollTo({ top: 0, behavior: 'smooth' });
        }

        function checkAnswers() {
            let score = 0;
            currentQuizData.forEach((quiz, index) => {
                const explanation = document.getElementById(`explanation-${index}`);
                explanation.classList.remove('hidden');
                
                let userAnswer;
                if (quiz.type === 'multiple' || quiz.options) {
                    const selected = document.querySelector(`input[name="quiz-${index}"]:checked`);
                    userAnswer = selected ? parseInt(selected.value) : -1;
                    if (userAnswer === parseInt(quiz.answer)) score++;
                } else {
                    userAnswer = document.querySelector(`input[name="quiz-${index}"]`).value.trim();
                    if (userAnswer === quiz.answer) score++;
                }
            });

            const scoreDisplay = document.getElementById('score-display');
            const message = document.getElementById('result-message');
            scoreDisplay.innerText = `${score} / ${currentQuizData.length}`;
            
            if (score === currentQuizData.length) message.innerText = "대단해요! 책의 내용을 완벽하게 이해하셨군요! 🎉";
            else if (score >= currentQuizData.length / 2) message.innerText = "좋은 성적입니다! 조금만 더 꼼꼼히 읽어볼까요? 👍";
            else message.innerText = "다시 한번 내용을 살펴보면 더 잘할 수 있을 거예요! 💪";

            document.getElementById('result-modal').classList.remove('hidden');
        }

        function closeModal() {
            document.getElementById('result-modal').classList.add('hidden');
            window.scrollTo({ top: 0, behavior: 'smooth' });
        }

        function resetApp() {
            document.getElementById('book-content').value = '';
            document.getElementById('input-section').classList.remove('hidden');
            document.getElementById('quiz-section').classList.add('hidden');
            document.getElementById('result-modal').classList.add('hidden');
            window.scrollTo({ top: 0, behavior: 'smooth' });
        }
    </script>
</body>
</html>
