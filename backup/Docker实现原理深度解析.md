<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>交互式限流算法指南</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Noto+Sans+SC:wght@300;400;500;700&display=swap" rel="stylesheet">
    <!-- Chosen Palette: Calm Harmony -->
    <!-- Application Structure Plan: The application is structured into thematic sections to guide the user from basic concepts to deep-dive comparisons and practical implementation. It starts with a 'What & Why' intro, moves to an 'Algorithm Explorer' with interactive visualizations (the core of the app), then to a 'Comparative Analysis' section with a radar chart for easy comparison, and finally an 'Implementation Guide'. This task-oriented flow (Understand -> Explore -> Compare -> Implement) was chosen over a linear report structure to enhance learning and usability, allowing users to jump to the section most relevant to them. -->
    <!-- Visualization & Content Choices: Report info is mapped to interactive elements. 1) Algorithm Principles -> Goal: Explore -> Viz: Custom Canvas Animations -> Interaction: Watch simulated request flows -> Justification: Visually demonstrating the core logic is more intuitive than text alone. 2) Algorithm Comparison -> Goal: Compare -> Viz: Radar Chart (Chart.js) -> Interaction: Hover to see details -> Justification: A radar chart provides an excellent multi-axial view to compare algorithms across key metrics like complexity and burst handling. 3) Implementation Libraries -> Goal: Inform -> Viz: Tabbed content cards -> Interaction: Click tabs for different languages -> Justification: Organizes a large amount of information cleanly. -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
    <style>
        body {
            font-family: 'Noto Sans SC', sans-serif;
            background-color: #f8fafc; /* neutral-50 */
            color: #1f2937; /* neutral-800 */
        }
        .chart-container {
            position: relative;
            width: 100%;
            max-width: 600px;
            margin-left: auto;
            margin-right: auto;
            height: 300px;
            max-height: 400px;
        }
        @media (min-width: 768px) {
            .chart-container {
                height: 400px;
            }
        }
        .tab.active {
            border-color: #3b82f6; /* blue-500 */
            background-color: #eff6ff; /* blue-50 */
            color: #2563eb; /* blue-600 */
            font-weight: 500;
        }
        .section-title {
            font-size: 2.25rem;
            font-weight: 700;
            text-align: center;
            margin-bottom: 1rem;
            color: #111827; /* gray-900 */
        }
        .section-subtitle {
            font-size: 1.125rem;
            text-align: center;
            max-width: 48rem;
            margin: 0 auto 3rem auto;
            color: #4b5563; /* gray-600 */
        }
        .card {
            background-color: white;
            border-radius: 0.75rem;
            box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
            transition: transform 0.2s ease-in-out, box-shadow 0.2s ease-in-out;
        }
        .card:hover {
            transform: translateY(-4px);
            box-shadow: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -2px rgb(0 0 0 / 0.1);
        }
        .icon-placeholder {
            font-size: 2rem;
            line-height: 1;
        }
    </style>
</head>
<body class="antialiased">

    <header class="bg-white py-8">
        <div class="container mx-auto px-4 text-center">
            <h1 class="text-4xl md:text-5xl font-bold text-gray-900 mb-2">交互式限流算法指南</h1>
            <p class="text-lg text-gray-600 max-w-3xl mx-auto">一个将复杂的系统保护机制转化为直观、可交互体验的探索平台。</p>
        </div>
    </header>

    <main class="container mx-auto px-4 py-12">
        
        <section id="intro" class="mb-20">
            <h2 class="section-title">什么是限流？为何如此重要？</h2>
            <p class="section-subtitle">
                限流 (Rate Limiting) 是现代系统设计的基石，它通过控制请求速率来保护服务免于过载、确保资源公平分配并抵御恶意攻击。这不仅是防御策略，更是主动的资源管理，直接关系到服务的稳定性和用户体验。
            </p>
            <div class="grid md:grid-cols-3 gap-8 text-center">
                <div class="card p-6">
                    <div class="icon-placeholder mb-4 text-blue-500">🛡️</div>
                    <h3 class="text-xl font-semibold mb-2">防止系统过载</h3>
                    <p class="text-gray-600">避免因流量洪峰导致的服务响应缓慢或崩溃，保障系统核心稳定性。</p>
                </div>
                <div class="card p-6">
                    <div class="icon-placeholder mb-4 text-green-500">⚖️</div>
                    <h3 class="text-xl font-semibold mb-2">确保公平使用</h3>
                    <p class="text-gray-600">防止少数用户滥用资源，确保所有合法用户都能公平地访问服务。</p>
                </div>
                <div class="card p-6">
                    <div class="icon-placeholder mb-4 text-red-500">💥</div>
                    <h3 class="text-xl font-semibold mb-2">抵御恶意攻击</h3>
                    <p class="text-gray-600">有效防御DDoS、暴力破解和恶意爬虫等攻击，增强系统安全性。</p>
                </div>
            </div>
        </section>

        <section id="explorer" class="mb-20">
            <h2 class="section-title">算法浏览器</h2>
            <p class="section-subtitle">
                选择一个算法，通过交互式动画和简洁的说明来深入理解其工作原理。观察请求如何被处理、允许或拒绝，直观感受每种策略的独特之处。
            </p>

            <div class="flex justify-center mb-8 border-b border-gray-200">
                <div id="tabs-container" class="flex flex-wrap -mb-px">
                </div>
            </div>

            <div id="algorithm-content" class="grid lg:grid-cols-2 gap-12 items-start">
                <div class="card p-6 h-full">
                    <h3 class="text-2xl font-bold mb-4" id="algorithm-title"></h3>
                    <div class="w-full h-64 bg-gray-100 rounded-lg mb-4">
                       <canvas id="algorithm-visualizer"></canvas>
                    </div>
                    <div class="text-center text-sm text-gray-500 mb-4">
                        <button id="reset-animation-btn" class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 transition">重新开始动画</button>
                    </div>
                     <p class="mb-4 text-gray-700" id="algorithm-description"></p>
                </div>
                <div class="space-y-6">
                    <div class="card p-6">
                        <h4 class="text-xl font-semibold mb-3">✅ 优点</h4>
                        <ul id="algorithm-pros" class="list-disc list-inside space-y-2 text-gray-700"></ul>
                    </div>
                    <div class="card p-6">
                        <h4 class="text-xl font-semibold mb-3">❌ 缺点</h4>
                        <ul id="algorithm-cons" class="list-disc list-inside space-y-2 text-gray-700"></ul>
                    </div>
                     <div class="card p-6">
                        <h4 class="text-xl font-semibold mb-3">🎯 典型适用场景</h4>
                        <ul id="algorithm-use-cases" class="list-disc list-inside space-y-2 text-gray-700"></ul>
                    </div>
                </div>
            </div>
        </section>
        
        <section id="comparison" class="mb-20">
            <h2 class="section-title">对比分析</h2>
            <p class="section-subtitle">
                没有完美的算法，只有最适合的。通过下面的雷达图直观对比各算法在关键维度上的表现，帮助您根据业务场景的特定需求做出权衡和选择。
            </p>
            <div class="card p-6">
                <div class="chart-container">
                    <canvas id="comparison-chart"></canvas>
                </div>
            </div>
        </section>

        <section id="implementation">
            <h2 class="section-title">实现与工具</h2>
             <p class="section-subtitle">
                了解限流在系统中的不同实现位置，并探索各大编程语言中成熟的库与框架，快速将理论付诸实践。
            </p>
            <div class="grid md:grid-cols-2 lg:grid-cols-4 gap-8">
                 <div class="card p-6">
                    <h3 class="text-xl font-semibold mb-2">Java</h3>
                    <ul class="list-disc list-inside space-y-1 text-gray-600">
                        <li>Google Guava RateLimiter</li>
                        <li>Bucket4j</li>
                        <li>Resilience4j</li>
                    </ul>
                </div>
                 <div class="card p-6">
                    <h3 class="text-xl font-semibold mb-2">Python</h3>
                     <ul class="list-disc list-inside space-y-1 text-gray-600">
                        <li>Flask-Limiter</li>
                        <li>requests-ratelimiter</li>
                    </ul>
                </div>
                 <div class="card p-6">
                    <h3 class="text-xl font-semibold mb-2">Go</h3>
                    <ul class="list-disc list-inside space-y-1 text-gray-600">
                        <li>golang.org/x/time/rate</li>
                        <li>sethvargo/go-limiter</li>
                    </ul>
                </div>
                 <div class="card p-6">
                    <h3 class="text-xl font-semibold mb-2">Node.js</h3>
                    <ul class="list-disc list-inside space-y-1 text-gray-600">
                        <li>express-rate-limit</li>
                        <li>dynamic-rate-limiter</li>
                        <li>NestJS Throttler</li>
                    </ul>
                </div>
            </div>
        </section>

    </main>
    
    <footer class="bg-gray-800 text-white mt-20 py-8">
        <div class="container mx-auto px-4 text-center text-gray-400">
            <p>基于源报告构建的交互式应用。</p>
            <p>旨在提供更直观的学习体验。</p>
        </div>
    </footer>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const ALGORITHMS = {
                'fixed-window': {
                    name: '固定窗口计数器',
                    description: '在固定的时间窗口内计算请求数。简单直观，但会在窗口边界产生突发问题。',
                    pros: ['实现简单，易于理解和维护。', '资源消耗低，只需一个计数器。'],
                    cons: ['窗口边缘的突发流量问题，可能导致瞬间请求量翻倍。', '流量控制不够平滑，可能误伤合法用户。'],
                    useCases: ['流量可预测的简单场景。', '对突发流量不敏感的内部API。']
                },
                'sliding-log': {
                    name: '滑动窗口日志',
                    description: '在滑动的时间窗口内记录每个请求的时间戳。精确控制速率，但内存消耗较大。',
                    pros: ['精确控制请求速率，有效缓解了窗口边界问题。', '流量控制非常平滑。'],
                    cons: ['需要存储每个请求的时间戳，内存占用高。', '实现相对复杂，有一定计算开销。'],
                    useCases: ['需要精确速率控制的高流量应用。', '对公平性要求高的场景，如金融交易API。']
                },
                'sliding-counter': {
                    name: '滑动窗口计数器',
                    description: '滑动窗口日志的优化版，结合了固定窗口和滑动窗口的优点，平衡了性能与资源。',
                    pros: ['内存效率高，比日志法节省大量空间。', '很好地处理了突发流量，性能与资源平衡。'],
                    cons: ['实现比固定窗口复杂。', '存在一定不准确性（假设前一窗口请求均匀分布）。'],
                    useCases: ['需要平衡性能和资源效率的可伸缩API。', '大型分布式系统中的通用限流方案。']
                },
                'token-bucket': {
                    name: '令牌桶',
                    description: '以固定速率向桶中放入令牌，请求消耗令牌。能有效应对突发流量。',
                    pros: ['允许并能平滑处理突发流量。', '高效利用网络带宽，灵活性高。'],
                    cons: ['令牌耗尽时新请求会延迟。', '不适合需要完美稳定速率的场景。'],
                    useCases: ['视频流（允许快速缓冲）、文件上传。', '电商秒杀等需要应对瞬时高并发的场景。']
                },
                'leaky-bucket': {
                    name: '漏桶',
                    description: '请求进入桶中排队，以恒定速率流出。能强制平滑输出流量，但会丢弃突发请求。',
                    pros: ['确保平滑、恒定的输出流量。', '实现简单，内存效率高。'],
                    cons: ['无法有效处理突发流量，多余请求会被丢弃。', '无法充分利用瞬时可用带宽。'],
                    useCases: ['VoIP、实时流媒体等对流量平滑度要求极高的应用。', 'ISP网络提供商进行流量整形。']
                },
            };

            const COMPARISON_DATA = {
                labels: ['突发处理能力', '流量平滑度', '资源利用效率', '实现复杂度', '精确度'],
                datasets: [
                    {
                        label: '固定窗口',
                        data: [1, 2, 5, 5, 2],
                        borderColor: 'rgba(255, 99, 132, 1)',
                        backgroundColor: 'rgba(255, 99, 132, 0.2)',
                    },
                    {
                        label: '滑动窗口日志',
                        data: [4, 5, 2, 2, 5],
                        borderColor: 'rgba(54, 162, 235, 1)',
                        backgroundColor: 'rgba(54, 162, 235, 0.2)',
                    },
                     {
                        label: '滑动窗口计数器',
                        data: [4, 4, 4, 3, 4],
                        borderColor: 'rgba(255, 206, 86, 1)',
                        backgroundColor: 'rgba(255, 206, 86, 0.2)',
                    },
                    {
                        label: '令牌桶',
                        data: [5, 3, 4, 3, 4],
                        borderColor: 'rgba(75, 192, 192, 1)',
                        backgroundColor: 'rgba(75, 192, 192, 0.2)',
                    },
                    {
                        label: '漏桶',
                        data: [1, 5, 4, 4, 3],
                        borderColor: 'rgba(153, 102, 255, 1)',
                        backgroundColor: 'rgba(153, 102, 255, 0.2)',
                    }
                ]
            };

            let currentState = {
                activeAlgorithm: 'fixed-window',
                animationFrameId: null,
            };

            const tabsContainer = document.getElementById('tabs-container');
            const titleEl = document.getElementById('algorithm-title');
            const descriptionEl = document.getElementById('algorithm-description');
            const prosEl = document.getElementById('algorithm-pros');
            const consEl = document.getElementById('algorithm-cons');
            const useCasesEl = document.getElementById('algorithm-use-cases');
            const visualizerCanvas = document.getElementById('algorithm-visualizer');
            const comparisonCanvas = document.getElementById('comparison-chart');
            const resetBtn = document.getElementById('reset-animation-btn');

            const ctx = visualizerCanvas.getContext('2d');
            
            function renderTabs() {
                tabsContainer.innerHTML = '';
                Object.keys(ALGORITHMS).forEach(id => {
                    const tab = document.createElement('button');
                    tab.dataset.id = id;
                    tab.textContent = ALGORITHMS[id].name;
                    tab.className = `tab text-lg px-4 py-3 border-b-2 font-medium text-gray-500 hover:text-gray-700 hover:border-gray-300 transition-colors duration-200 ease-in-out ${currentState.activeAlgorithm === id ? 'active' : 'border-transparent'}`;
                    tab.addEventListener('click', () => {
                        currentState.activeAlgorithm = id;
                        render();
                    });
                    tabsContainer.appendChild(tab);
                });
            }

            function renderContent() {
                const algorithm = ALGORITHMS[currentState.activeAlgorithm];
                titleEl.textContent = algorithm.name;
                descriptionEl.textContent = algorithm.description;

                prosEl.innerHTML = algorithm.pros.map(p => `<li>${p}</li>`).join('');
                consEl.innerHTML = algorithm.cons.map(c => `<li>${c}</li>`).join('');
                useCasesEl.innerHTML = algorithm.useCases.map(u => `<li>${u}</li>`).join('');

                startAnimation();
            }

            function renderComparisonChart() {
                new Chart(comparisonCanvas, {
                    type: 'radar',
                    data: COMPARISON_DATA,
                    options: {
                        maintainAspectRatio: false,
                        scales: {
                            r: {
                                angleLines: {
                                    display: true,
                                    color: 'rgba(0, 0, 0, 0.1)'
                                },
                                suggestedMin: 0,
                                suggestedMax: 5,
                                pointLabels: {
                                    font: {
                                        size: 14,
                                        family: "'Noto Sans SC', sans-serif"
                                    }
                                },
                                ticks: {
                                   stepSize: 1
                                }
                            }
                        },
                        plugins: {
                            legend: {
                                position: 'top',
                                labels: {
                                     font: {
                                        size: 12,
                                        family: "'Noto Sans SC', sans-serif"
                                    }
                                }
                            }
                        }
                    }
                });
            }

            function startAnimation() {
                if (currentState.animationFrameId) {
                    cancelAnimationFrame(currentState.animationFrameId);
                }
                const animator = getAnimator(currentState.activeAlgorithm);
                animator();
            }

            function getAnimator(algorithmId) {
                switch(algorithmId) {
                    case 'fixed-window': return animateFixedWindow;
                    case 'sliding-log': return animateSlidingLog;
                    case 'sliding-counter': return animateSlidingCounter;
                    case 'token-bucket': return animateTokenBucket;
                    case 'leaky-bucket': return animateLeakyBucket;
                    default: return animateFixedWindow;
                }
            }

            function render() {
                renderTabs();
                renderContent();
            }

            resetBtn.addEventListener('click', startAnimation);
            
            // Animation logic starts here
            let requests = [];
            const MAX_REQUESTS = 50;

            function drawRequest(req) {
                ctx.beginPath();
                ctx.arc(req.x, req.y, 5, 0, Math.PI * 2);
                ctx.fillStyle = req.color;
                ctx.fill();
            }

            function animateFixedWindow() {
                let frameCount = 0;
                let reqCountInWindow = 0;
                const windowDuration = 300; // frames
                const limit = 5;
                const w = visualizerCanvas.width;
                const h = visualizerCanvas.height;
                requests = [];

                function loop() {
                    frameCount++;
                    if (frameCount > windowDuration * 2) {
                        frameCount = 0;
                        reqCountInWindow = 0;
                        requests = [];
                    }
                    if (frameCount % windowDuration < 5) {
                         reqCountInWindow = 0;
                    }
                    
                    ctx.clearRect(0, 0, w, h);
                    
                    // Draw window
                    const windowIndex = Math.floor(frameCount / windowDuration);
                    const windowX = windowIndex * (w / 2);
                    ctx.fillStyle = 'rgba(54, 162, 235, 0.1)';
                    ctx.fillRect(windowX, 0, w/2, h);
                    ctx.strokeStyle = 'rgba(54, 162, 235, 0.5)';
                    ctx.strokeRect(windowX, 0, w/2, h);
                    
                    // Generate new request
                    if (Math.random() < 0.1 && requests.length < MAX_REQUESTS) {
                        const isAllowed = reqCountInWindow < limit;
                        if (isAllowed) reqCountInWindow++;
                        requests.push({
                            x: 0,
                            y: h/2,
                            vx: 2,
                            color: isAllowed ? 'green' : 'red',
                            isAllowed: isAllowed
                        });
                    }

                    // Update and draw requests
                    requests.forEach(req => {
                        if (req.isAllowed) {
                           req.x += req.vx;
                        } else {
                           req.x += req.vx / 4;
                           req.y += (Math.random() - 0.5) * 2; // jitter
                        }
                        drawRequest(req);
                    });
                    requests = requests.filter(req => req.x < w);

                    // Draw text
                    ctx.fillStyle = 'black';
                    ctx.font = '16px Noto Sans SC';
                    ctx.fillText(`窗口 ${windowIndex+1}`, windowX + 10, 30);
                    ctx.fillText(`计数: ${reqCountInWindow} / ${limit}`, windowX + 10, 50);

                    currentState.animationFrameId = requestAnimationFrame(loop);
                }
                loop();
            }

            function animateSlidingLog() {
                let timestamps = [];
                const windowSize = 5000; // ms
                const limit = 8;
                const w = visualizerCanvas.width;
                const h = visualizerCanvas.height;
                requests = [];
                let startTime = Date.now();

                function loop() {
                    const now = Date.now();
                    ctx.clearRect(0, 0, w, h);

                    timestamps = timestamps.filter(t => now - t < windowSize);

                    if (Math.random() < 0.1 && requests.length < MAX_REQUESTS) {
                        const isAllowed = timestamps.length < limit;
                        if (isAllowed) timestamps.push(now);
                         requests.push({
                            x: 0,
                            y: h/2 + (Math.random() - 0.5) * 30,
                            vx: 2,
                            color: isAllowed ? 'green' : 'red',
                            isAllowed: isAllowed
                        });
                    }

                    requests.forEach(req => {
                        if (req.isAllowed) req.x += req.vx;
                        else req.x += req.vx/4;
                        drawRequest(req);
                    });
                    requests = requests.filter(r => r.x < w);

                    // Draw window visualization
                    ctx.strokeStyle = '#36A2EB';
                    ctx.lineWidth = 2;
                    ctx.strokeRect(w/2 - 150, h-40, 300, 30);

                    timestamps.forEach(t => {
                        const age = now - t;
                        const xPos = (w/2 - 150) + 300 * (1 - age / windowSize);
                        ctx.fillStyle = '#36A2EB';
                        ctx.fillRect(xPos - 2, h-35, 4, 20);
                    });

                    ctx.fillStyle = 'black';
                    ctx.font = '16px Noto Sans SC';
                    ctx.fillText(`过去 ${windowSize/1000}s 内的请求日志`, 10, 30);
                    ctx.fillText(`计数: ${timestamps.length} / ${limit}`, 10, 50);

                    currentState.animationFrameId = requestAnimationFrame(loop);
                }
                loop();
            }
            
            function animateSlidingCounter() {
                 let prevCount = 0;
                 let currentCount = 0;
                 let frameCount = 0;
                 const windowDuration = 300;
                 const limit = 8;
                 const w = visualizerCanvas.width;
                 const h = visualizerCanvas.height;

                function loop() {
                    frameCount++;
                     if(frameCount >= windowDuration) {
                         frameCount = 0;
                         prevCount = currentCount;
                         currentCount = 0;
                     }
                    
                    ctx.clearRect(0, 0, w, h);

                    const weight = frameCount / windowDuration;
                    const estimatedCount = Math.floor(prevCount * (1 - weight) + currentCount);

                    if (Math.random() < 0.1) {
                         const isAllowed = estimatedCount < limit;
                         if (isAllowed) currentCount++;
                         requests.push({
                            y: 10,
                            vy: 2,
                            color: isAllowed ? 'green' : 'red',
                            isAllowed
                        });
                    }

                    requests.forEach(req => {
                        if (req.isAllowed) req.y += req.vy;
                        drawRequest({x: w/2, y: req.y, color: req.color});
                    });
                    requests = requests.filter(r => r.y < h);

                    // Draw Windows
                    ctx.fillStyle = 'rgba(255, 206, 86, 0.2)';
                    ctx.fillRect(w/4, h-80, w/4, 60);
                    ctx.strokeRect(w/4, h-80, w/4, 60);
                    ctx.fillStyle = 'black';
                    ctx.fillText(`前一窗口: ${prevCount}`, w/4 + 10, h-50);
                    
                    ctx.fillStyle = 'rgba(255, 206, 86, 0.4)';
                    ctx.fillRect(w/2, h-80, w/4, 60);
                    ctx.strokeRect(w/2, h-80, w/4, 60);
                    ctx.fillStyle = 'black';
                    ctx.fillText(`当前窗口: ${currentCount}`, w/2 + 10, h-50);

                    // Draw Progress bar
                    ctx.fillStyle = '#ccc';
                    ctx.fillRect(w/2, h-100, w/4, 10);
                    ctx.fillStyle = '#FFCE56';
                    ctx.fillRect(w/2, h-100, (w/4) * weight, 10);

                    ctx.font = '16px Noto Sans SC';
                    ctx.fillText(`滑动窗口近似计数: ${estimatedCount} / ${limit}`, 10, 30);
                    ctx.fillText(`计算公式: ${prevCount} * ${(1-weight).toFixed(2)} + ${currentCount} = ${estimatedCount}`, 10, 50);
                    currentState.animationFrameId = requestAnimationFrame(loop);
                }
                loop();
            }

            function animateTokenBucket() {
                let tokens = 5;
                const capacity = 10;
                const rate = 0.05; // tokens per frame
                const w = visualizerCanvas.width;
                const h = visualizerCanvas.height;
                requests = [];

                function loop() {
                    tokens += rate;
                    if (tokens > capacity) tokens = capacity;

                    ctx.clearRect(0, 0, w, h);

                    if (Math.random() < 0.1 && requests.length < MAX_REQUESTS) {
                         const isAllowed = tokens >= 1;
                         if (isAllowed) tokens--;
                         requests.push({
                            x: 0,
                            y: h - 20,
                            vx: 2,
                            color: isAllowed ? 'green' : 'red',
                            isAllowed
                        });
                    }

                    requests.forEach(req => {
                        req.x += req.vx;
                        drawRequest(req);
                    });
                    requests = requests.filter(r => r.x < w);

                    // Draw bucket
                    const bucketX = w/2 - 50, bucketY = 50, bucketW = 100, bucketH = 150;
                    ctx.strokeStyle = '#4BC0C0';
                    ctx.lineWidth = 2;
                    ctx.beginPath();
                    ctx.moveTo(bucketX, bucketY);
                    ctx.lineTo(bucketX, bucketY + bucketH);
                    ctx.lineTo(bucketX + bucketW, bucketY + bucketH);
                    ctx.lineTo(bucketX + bucketW, bucketY);
                    ctx.stroke();

                    // Draw tokens
                    ctx.fillStyle = '#4BC0C0';
                    for (let i = 0; i < Math.floor(tokens); i++) {
                        const x = bucketX + 15 + (i % 5) * 15;
                        const y = bucketY + bucketH - 15 - Math.floor(i / 5) * 15;
                        ctx.beginPath();
                        ctx.arc(x, y, 5, 0, Math.PI * 2);
                        ctx.fill();
                    }
                    
                    ctx.fillStyle = 'black';
                    ctx.font = '16px Noto Sans SC';
                    ctx.fillText(`令牌数: ${Math.floor(tokens)} / ${capacity}`, 10, 30);

                    currentState.animationFrameId = requestAnimationFrame(loop);
                }
                loop();
            }
            
            function animateLeakyBucket() {
                 let waterLevel = 0;
                 const capacity = 10;
                 const leakRate = 0.03; // requests per frame
                 const w = visualizerCanvas.width;
                 const h = visualizerCanvas.height;
                 requests = [];
                 
                 function loop() {
                    waterLevel -= leakRate;
                    if (waterLevel < 0) waterLevel = 0;
                    
                    ctx.clearRect(0, 0, w, h);

                    if (Math.random() < 0.1) {
                         const canQueue = waterLevel < capacity;
                         if (canQueue) waterLevel++;
                         requests.push({
                            x: Math.random() * w,
                            y: 0,
                            vy: 2,
                            color: canQueue ? '#9966FF' : 'red',
                            isQueued: canQueue,
                            isLeaked: false
                        });
                    }

                    // Leaking requests
                    if (waterLevel > 0) {
                        const leakedReq = requests.find(r => r.isQueued && !r.isLeaked);
                        if (leakedReq) {
                           leakedReq.isLeaked = true;
                           leakedReq.color = 'green';
                        }
                    }

                    requests.forEach(req => {
                        if (!req.isQueued) { // Rejected
                            req.y -= req.vy;
                        } else if (!req.isLeaked) { // Queued
                             req.y += req.vy / 2;
                             if(req.y > 60) req.y = 60;
                        } else { // Leaked
                             req.y += req.vy;
                        }
                        drawRequest(req);
                    });
                    requests = requests.filter(r => r.y < h && r.y > -10);

                    // Draw bucket
                    const bucketX = w/2 - 75, bucketY = 50, bucketW = 150, bucketH = 100;
                    ctx.strokeStyle = '#9966FF';
                    ctx.lineWidth = 2;
                    ctx.strokeRect(bucketX, bucketY, bucketW, bucketH);
                    
                    // Draw water level
                    const waterHeight = (waterLevel / capacity) * bucketH;
                    ctx.fillStyle = 'rgba(153, 102, 255, 0.3)';
                    ctx.fillRect(bucketX, bucketY + bucketH - waterHeight, bucketW, waterHeight);
                    
                    // Draw leak
                    ctx.fillStyle = 'green';
                    ctx.fillRect(w/2 - 2, bucketY + bucketH, 4, 10);
                    
                    ctx.fillStyle = 'black';
                    ctx.font = '16px Noto Sans SC';
                    ctx.fillText(`队列长度: ${Math.floor(waterLevel)} / ${capacity}`, 10, 30);
                    
                    currentState.animationFrameId = requestAnimationFrame(loop);
                 }
                 loop();
            }

            // Initial render
            render();
            renderComparisonChart();
        });
    </script>
</body>
</html>
