document.addEventListener('DOMContentLoaded', function() {
    const video = document.getElementById('videoElement');
    const bufferLengthDisplay = document.getElementById('buffer-length');
    const stallCountDisplay = document.getElementById('stall-count');
    const stallDurationDisplay = document.getElementById('stall-duration');
    const droppedFramesDisplay = document.getElementById('dropped-frames');
    const qualityDropdown = document.getElementById('quality-dropdown');

    // 清晰度配置
    const QUALITY_LEVELS = [
        { url: 'http://120.25.174.232:8080/live/livestream_low.flv', bitrate: 1000, quality: '360P' },
        { url: 'http://120.25.174.232:8080/live/livestream_mid.flv', bitrate: 2000, quality: '540P' },
        { url: 'http://120.25.174.232:8080/live/livestream_high.flv', bitrate: 3000, quality: '720P' }
    ];

    // 状态变量
    let stallCount = 0;
    let loader = null;
    let totalStallDuration = 0;
    let stallStartTime = null;
    let currentQualityIndex = 1;
    let flvPlayer = null;
    let bitrateSwitchInterval = null;
    let lastSwitchTime = 0;
    const SWITCH_COOLDOWN = 5000;
    let stallRecords = [];
    let isAutoMode = true;
    let bitrateHistory = []; // 码率历史记录
    let isRetrying = false; //重试状态标识

    if (flvjs.isSupported()) {
        // 初始化播放器（分拆为销毁和新建逻辑）
        function initPlayer(url) {
            const cacheBusterUrl = `${url}?t=${Date.now()}`;

            // 尝试重用播放器实例
            if (flvPlayer && flvPlayer._config) {
                try {
                    flvPlayer.pause();
                    flvPlayer.unload();
                    flvPlayer.detachMediaElement();

                    // 直接修改配置参数
                    flvPlayer._config.url = cacheBusterUrl;
                    flvPlayer.attachMediaElement(video);
                    flvPlayer.load();

                    // 优化缓冲配置
                    flvPlayer._config.enableStashBuffer = true;
                    flvPlayer._config.stashInitialSize = 64;

                    video.muted = true;
                    video.play().catch(err => console.log('自动播放失败:', err));
                    console.log('[DEBUG] 播放器实例已重用');
                    return;
                } catch (e) {
                    console.warn('播放器重用失败，创建新实例:', e);
                }
            }
            if (flvPlayer) {
                flvPlayer.pause();
                flvPlayer.unload();
                flvPlayer.detachMediaElement();
                flvPlayer.destroy();
                flvPlayer = null;
                console.log('[DEBUG] 旧播放器实例已销毁');
                setTimeout(() => initNewPlayer(url), 100); // 延迟确保垃圾回收
            } else {
                initNewPlayer(url);
            }
        }

        function initNewPlayer(url) {
            const cacheBusterUrl = `${url}?t=${Date.now()}`;

            flvPlayer = flvjs.createPlayer({
                type: 'flv',
                url: cacheBusterUrl,
                isLive: true
            }, {
                enableWorker: false,
                enableStashBuffer: true,
                stashInitialSize: 32, // 增大初始缓冲
                lazyLoad: true,
                autoCleanupSourceBuffer: true,
                enableStats: true,
                statsCallback: (stats) => {
                    const droppedFrames = stats.droppedFrames || 0;
                    droppedFramesDisplay.textContent = droppedFrames;
                }
            });


            // 监听加载器实例创建事件
            flvPlayer.on(flvjs.Events.LOADER_CREATED, (_, loaders) => {
                loader = loaders.loader;
                console.log('[DEBUG] Loader 已初始化:', loader);
            });

            flvPlayer.attachMediaElement(video);
            flvPlayer.load();

            flvPlayer.on(flvjs.Events.ERROR, (errorType, errorDetail, errorInfo) => {
                console.error('FLV.js Error:', errorType, errorDetail, errorInfo);

                // 处理网络类错误
                if ([flvjs.ErrorTypes.NETWORK_ERROR,
                        flvjs.ErrorTypes.LOADER_ERROR
                    ].includes(errorType)) {
                    console.log('[ERROR] 加载失败，启动重试机制...');

                    // 清除现有定时器
                    clearInterval(bitrateSwitchInterval);
                    clearInterval(retryInterval);

                    // 启动随机重试机制
                    retryInterval = setInterval(() => {
                        const oldIndex = currentQualityIndex;
                        let newIndex;

                        // 确保切换到不同清晰度
                        do {
                            newIndex = Math.floor(Math.random() * QUALITY_LEVELS.length);
                        } while (newIndex === oldIndex);

                        console.log(`[RETRY] 尝试切换到: ${QUALITY_LEVELS[newIndex].quality}`);
                        currentQualityIndex = newIndex;
                        initPlayer(QUALITY_LEVELS[newIndex].url);
                    }, 5000);
                }
            });

            // 播放器加载完成后触发
            flvPlayer.on(flvjs.Events.MEDIA_INFO, () => {
                console.log('[DEBUG] 播放器加载完成，准备启动ABR');

                // 成功加载后清除重试机制
                clearInterval(retryInterval);

                if (isAutoMode) {
                    isRetrying = false;
                    clearInterval(retryInterval);
                    startABR();
                }
            });

            // 尝试自动播放（静音模式）
            video.muted = true;
            video.play().catch(err => {
                console.log('Autoplay blocked, show play button');
                showPlayButton();
            });

            // 更新显示信息
            document.getElementById('current-bitrate').textContent = QUALITY_LEVELS[currentQualityIndex].bitrate;
            document.getElementById('current-quality').textContent = QUALITY_LEVELS[currentQualityIndex].quality;
        }

        // 初始化下拉菜单
        function setupQualitySelector() {
            qualityDropdown.addEventListener('change', (e) => {
                const value = e.target.value;

                if (value === 'auto') {
                    enableAutoMode();
                } else {
                    const qualityMap = { '360': 0, '540': 1, '720': 2 };
                    const index = qualityMap[value];
                    if (index !== undefined) {
                        disableAutoMode(index);
                    }
                }
            });
        }

        function enableAutoMode() {
            isAutoMode = true;
            clearInterval(bitrateSwitchInterval);
            // 若 loader 未初始化，延迟启动
            if (!loader) {
                console.log('[DEBUG] 等待 loader 初始化...');
                setTimeout(() => enableAutoMode(), 100);
                return;
            }

            startABR();
            checkBitrate(); // 立即触发一次检查
            updateQualityDisplay();
            console.log('[ACTION] 切换到自适应模式');
        }

        function disableAutoMode(index) {
            isAutoMode = false;
            clearInterval(retryInterval);
            clearInterval(bitrateSwitchInterval);
            switchQuality(index);
            console.log(`[ACTION] 切换到手动模式: ${QUALITY_LEVELS[index].quality}`);
        }


        // 更新缓冲区长度
        video.addEventListener('timeupdate', () => {
            if (video.buffered.length > 0) {
                const bufferEnd = video.buffered.end(video.buffered.length - 1);
                const bufferLength = bufferEnd - video.currentTime;
                bufferLengthDisplay.textContent = bufferLength.toFixed(2);
            }
        });

        // 卡顿开始时间记录
        video.addEventListener('waiting', () => {
            if (!stallStartTime) {
                stallStartTime = performance.now();
                stallCount++;
                stallCountDisplay.textContent = stallCount;
            }
        });

        // 卡顿结束时间计算
        video.addEventListener('playing', () => {
            if (stallStartTime) {
                const stallEndTime = performance.now();
                const duration = (stallEndTime - stallStartTime) / 1000;
                totalStallDuration += duration;
                stallDurationDisplay.textContent = totalStallDuration.toFixed(2);
                stallRecords.push({
                    timestamp: Date.now(),
                    duration: duration
                });
                stallStartTime = null;
            }
        });

        // 获取有效缓冲区长度
        function getBufferLength() {
            return video.buffered.length > 0 ?
                video.buffered.end(video.buffered.length - 1) - video.currentTime : 0;
        }

        // 获取近30秒卡顿时长
        function getRecentStallDuration() {
            const now = Date.now();
            return stallRecords
                .filter(record => now - record.timestamp < 30000)
                .reduce((sum, r) => sum + r.duration, 0);
        }

        // 更新显示信息（统一函数）
        function updateQualityDisplay() {
            const currentQuality = QUALITY_LEVELS[currentQualityIndex];
            document.getElementById('current-bitrate').textContent = currentQuality.bitrate;
            document.getElementById('current-quality').textContent =
                isAutoMode ? `${currentQuality.quality}` : currentQuality.quality;
        }

        // 修改错误处理函数
        function handleLoadFailure() {
            // 仅自动模式处理
            if (!isAutoMode) return;

            // 避免重复启动
            if (isRetrying) return;
            isRetrying = true;

            console.log('[RETRY] 进入自动恢复模式');
            clearInterval(bitrateSwitchInterval);

            // 立即执行首次切换
            performRandomSwitch();

            // 设置定时器
            retryInterval = setInterval(() => {
                performRandomSwitch();
            }, 5000);
        }

        // 独立随机切换函数
        function performRandomSwitch() {
            if (!isAutoMode) { // 双重检查模式状态
                clearInterval(retryInterval);
                return;
            }

            const oldIndex = currentQualityIndex;
            let newIndex;
            let candidates = [];

            // 权重配置：中间码率（2000kbps）概率50%，其他两个各25%
            if (oldIndex === 0) {
                candidates = [1, 1, 1, 2]; // 75%概率选中间
            } else if (oldIndex === 2) {
                candidates = [1, 1, 1, 0]; // 75%概率选中间
            } else {
                candidates = [0, 1, 1, 2]; // 旧是中间时保持低/高各50%
            }

            newIndex = candidates[Math.floor(Math.random() * candidates.length)];

            // 确保不重复当前索引（当旧是中间时才需要）
            if (oldIndex === 1 && newIndex === 1) {
                newIndex = Math.random() > 0.5 ? 0 : 2;
            }

            console.log(`[RETRY] 自动切换到: ${QUALITY_LEVELS[newIndex].quality}`);
            switchQuality(newIndex);
        }


        // 自适应码率逻辑（优化版）
        function checkBitrate() {
            console.log('[DEBUG] 检查码率 - loader状态:', loader ? '已加载' : '未加载'); // 新增调试日志
            // 仅在自动模式下执行检查
            if (!isAutoMode) return;

            // 当加载器未就绪时直接触发切换
            if (!loader || !loader._stats || !loader._stats.downloadSpeed) {
                console.log('[ABR] 播放器未就绪，启动紧急切换');
                handleLoadFailure();
                return; // 不再执行后续逻辑
            }

            // 冷却时间检查
            if (performance.now() - lastSwitchTime < SWITCH_COOLDOWN) {
                console.log('[DEBUG] 冷却时间内，跳过检查');
                return;
            }

            const currentLevel = QUALITY_LEVELS[currentQualityIndex];
            const bufferLength = getBufferLength();
            const measuredBitrate = loader._stats.downloadSpeed * 8; // bps
            const recentStall = getRecentStallDuration();

            // 滑动平均码率计算（最近5次）
            bitrateHistory.push(measuredBitrate);
            if (bitrateHistory.length > 5) bitrateHistory.shift();
            const avgBitrate = bitrateHistory.reduce((a, b) => a + b, 0) / bitrateHistory.length;
            //网络状态评分公式（示例）
            const networkScore = (avgBitrate / (currentLevel.bitrate * 1000)).toFixed(2);

            console.log(`[NETWORK] 码率计算（5秒刷新）：
                - 当前码率: ${currentLevel.bitrate}kbps
                - 测量平均码率: ${(avgBitrate/1000).toFixed(1)}kbps
                - 网络评分: ${networkScore}（≥1.2可升级，≤0.6需降级）
                - 缓冲区: ${bufferLength.toFixed(1)}s
                - 近期卡顿: ${recentStall.toFixed(1)}s`);

            // 紧急降级条件
            if (recentStall > 2 || bufferLength < 1) {
                if (currentQualityIndex > 0) {
                    switchQuality(currentQualityIndex - 1);
                    return;
                }
            }

            // 网络降级条件（平均码率不足60%）
            if (avgBitrate < currentLevel.bitrate * 1000 * 0.6) {
                if (currentQualityIndex > 0) {
                    switchQuality(currentQualityIndex - 1);
                    return;
                }
            }

            // 网络升级条件（缓冲充足且平均码率超过120%）
            if (bufferLength > 8 && avgBitrate > currentLevel.bitrate * 1000 * 1.2 && recentStall < 1) {
                if (currentQualityIndex < QUALITY_LEVELS.length - 1) {
                    switchQuality(currentQualityIndex + 1);
                }
            }
        }

        function switchQuality(newIndex) {
            lastSwitchTime = performance.now();
            currentQualityIndex = newIndex;
            loader = null; // 清除旧loader引用
            initPlayer(QUALITY_LEVELS[currentQualityIndex].url);
            stallRecords = []; // 重置卡顿记录

            // 获取当前质量信息
            const currentQuality = QUALITY_LEVELS[currentQualityIndex];

            // 统一更新显示
            document.getElementById('current-bitrate').textContent = currentQuality.bitrate;

            // 根据模式显示不同格式
            document.getElementById('current-quality').textContent = isAutoMode ?
                `${currentQuality.quality}` :
                currentQuality.quality;
            console.log(`切换到: ${QUALITY_LEVELS[currentQualityIndex].quality}`);
        }

        // 启动自适应码率逻辑
        function startABR() {
            clearInterval(bitrateSwitchInterval); // 清除旧定时器
            bitrateSwitchInterval = setInterval(() => {
                console.log('[DEBUG] 定时器触发');
                checkBitrate();
            }, 5000); // 5秒间隔
            console.log('[DEBUG] ABR定时器已启动');
        }

        // 清理资源
        window.addEventListener('beforeunload', () => {
            clearInterval(bitrateSwitchInterval);
            clearMonitor();
            if (flvPlayer) flvPlayer.destroy();
        });

        // 初始化
        setupQualitySelector();
        initPlayer(QUALITY_LEVELS[currentQualityIndex].url);
        qualityDropdown.value = 'auto'; // 默认自适应模式
        startABR();

        // 错误处理
        flvPlayer.on(flvjs.Events.ERROR, (errorType, errorDetail, errorInfo) => {
            console.error('FLV.js Error:', errorType, errorDetail, errorInfo);
        });
    } else {
        console.error('FLV.js is not supported in this browser.');
    }

});
