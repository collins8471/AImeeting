# 音訊診斷視窗實作指南

## 問題描述
使用者回報在上課模式下，麥克風持續有輸入，但一段時間後出現「Long silence detected」訊息並自動停止錄音。需要新增診斷工具來判斷問題根源。

## 解決方案：新增音訊診斷視窗

### 1. 在 `<style>` 區塊新增 CSS（第 80 行後）

```css
/* Diagnostic Panel Styles */
#diagnostic-canvas {
    width: 100%;
    height: 120px;
    background-color: #1e293b;
    border-radius: 8px;
}

.status-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 12px;
    border-radius: 9999px;
    font-size: 0.75rem;
    font-weight: 600;
}

.status-active { background-color: #d1fae5; color: #065f46; }
.status-stopped { background-color: #fee2e2; color: #991b1b; }
.status-error { background-color: #fef3c7; color: #92400e; }

.metric-box {
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 12px;
    padding: 16px;
}
```

### 2. 新增 Audio Diagnostic Modal HTML（第 279 行 Settings Modal 後）

```html
<!-- Audio Diagnostic Modal -->
<div id="diagnostic-modal" class="fixed inset-0 z-[70] bg-black/40 backdrop-blur-sm hidden flex items-center justify-center opacity-0 transition-opacity duration-300">
    <div class="bg-white rounded-2xl shadow-2xl w-full max-w-3xl max-h-[90vh] overflow-hidden scale-95 transition-transform duration-300">
        <!-- Header -->
        <div class="p-5 border-b border-slate-100 flex justify-between items-center bg-gradient-to-r from-yellow-50 to-amber-50">
            <div class="flex items-center gap-3">
                <div class="w-10 h-10 bg-yellow-100 text-yellow-600 rounded-xl flex items-center justify-center">
                    <i class="fa-solid fa-stethoscope text-xl"></i>
                </div>
                <div>
                    <h3 class="text-lg font-bold text-slate-800">🔍 音訊診斷工具</h3>
                    <p class="text-xs text-slate-500">即時監控麥克風與語音識別狀態</p>
                </div>
            </div>
            <button onclick="toggleDiagnostic()" class="w-8 h-8 rounded-full hover:bg-slate-200 flex items-center justify-center text-slate-500">
                <i class="fa-solid fa-xmark text-lg"></i>
            </button>
        </div>

        <!-- Content -->
        <div class="p-6 overflow-y-auto max-h-[calc(90vh-140px)]">
            <!-- Audio Waveform -->
            <div class="mb-6">
                <h4 class="text-sm font-bold text-slate-700 mb-3 flex items-center gap-2">
                    <i class="fa-solid fa-wave-square text-indigo-500"></i>
                    即時音訊波形
                </h4>
                <canvas id="diagnostic-canvas"></canvas>
                <div class="mt-2 flex justify-between items-center text-xs text-slate-500">
                    <span>音量強度: <span id="diag-volume" class="font-bold text-indigo-600">0%</span></span>
                    <span id="diag-wave-status" class="text-green-600 font-bold">● 監控中</span>
                </div>
            </div>

            <!-- Recognition Status -->
            <div class="mb-6 metric-box">
                <h4 class="text-sm font-bold text-slate-700 mb-3 flex items-center gap-2">
                    <i class="fa-solid fa-microphone text-emerald-500"></i>
                    Web Speech API 狀態
                </h4>
                <div class="space-y-2">
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">Recognition 狀態</span>
                        <span id="diag-recognition-state" class="status-badge status-stopped">STOPPED</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">上次 onstart 事件</span>
                        <span id="diag-last-start" class="text-sm text-slate-500 font-mono">未啟動</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">上次 onresult 事件</span>
                        <span id="diag-last-result" class="text-sm text-slate-500 font-mono">無資料</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">距離上次結果</span>
                        <span id="diag-time-since-result" class="text-sm font-bold text-amber-600">N/A</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">上次 onerror 事件</span>
                        <span id="diag-last-error" class="text-sm text-red-500 font-mono">無錯誤</span>
                    </div>
                </div>
            </div>

            <!-- Silence Detection -->
            <div class="mb-6 metric-box">
                <h4 class="text-sm font-bold text-slate-700 mb-3 flex items-center gap-2">
                    <i class="fa-solid fa-hourglass-half text-orange-500"></i>
                    靜音偵測監控
                </h4>
                <div class="space-y-2">
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">靜音超時設定</span>
                        <span class="text-sm font-bold text-slate-700">15 秒</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">倒數計時</span>
                        <span id="diag-silence-countdown" class="text-lg font-bold text-red-600">--</span>
                    </div>
                    <div class="mt-2 bg-slate-200 rounded-full h-2 overflow-hidden">
                        <div id="diag-silence-bar" class="bg-gradient-to-r from-green-500 to-red-500 h-full transition-all duration-300" style="width: 100%"></div>
                    </div>
                </div>
            </div>

            <!-- Device Info -->
            <div class="metric-box">
                <h4 class="text-sm font-bold text-slate-700 mb-3 flex items-center gap-2">
                    <i class="fa-solid fa-circle-info text-blue-500"></i>
                    裝置資訊
                </h4>
                <div class="space-y-2">
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">當前麥克風</span>
                        <span id="diag-device-name" class="text-sm font-bold text-slate-700">未知</span>
                    </div>
                    <div class="flex justify-between items-center">
                        <span class="text-sm text-slate-600">語言設定</span>
                        <span id="diag-language" class="text-sm font-bold text-slate-700">en-US</span>
                    </div>
                </div>
            </div>
        </div>

        <!-- Footer -->
        <div class="p-4 border-t border-slate-100 bg-slate-50 flex justify-between items-center">
            <button onclick="copyDiagnosticReport()" class="text-sm text-slate-600 hover:text-indigo-600 px-4 py-2 rounded-lg hover:bg-white transition-colors flex items-center gap-2">
                <i class="fa-regular fa-copy"></i>
                複製診斷報告
            </button>
            <button onclick="toggleDiagnostic()" class="px-5 py-2 bg-indigo-600 text-white rounded-xl hover:bg-indigo-700 font-bold transition-all">
                關閉
            </button>
        </div>
    </div>
</div>
```

### 3. 在 Header 工具列新增診斷按鈕（第 313 行 toggleAudioHelp() 按鈕後）

```html
<button onclick="toggleDiagnostic()" class="w-9 h-9 rounded-full bg-yellow-50 border border-yellow-200 text-yellow-600 hover:bg-yellow-100 transition-all flex items-center justify-center" title="音訊診斷">
    <i class="fa-solid fa-stethoscope"></i>
</button>
```

### 4. 新增診斷相關全域變數（第 410 行後）

```javascript
// Diagnostic Data
let diagnosticData = {
    lastStartTime: null,
    lastResultTime: null,
    lastErrorTime: null,
    lastErrorMessage: '',
    recognitionState: 'stopped',
    audioLevel: 0,
    silenceCountdown: 15
};
let diagnosticInterval = null;
let diagnosticCanvas, diagnosticCtx, diagnosticAnalyser;
```

### 5. 新增診斷視窗相關函式（在 script 區塊末尾，第 1120 行 scrollToBottom() 後）

```javascript
// === DIAGNOSTIC FUNCTIONS ===

function toggleDiagnostic() {
    toggleModal('diagnostic-modal');
    const modal = document.getElementById('diagnostic-modal');
    if (!modal.classList.contains('hidden')) {
        startDiagnostic();
    } else {
        stopDiagnostic();
    }
}

function startDiagnostic() {
    diagnosticCanvas = document.getElementById('diagnostic-canvas');
    if (!diagnosticCanvas) return;
    
    diagnosticCtx = diagnosticCanvas.getContext('2d');
    diagnosticCanvas.width = diagnosticCanvas.offsetWidth;
    diagnosticCanvas.height = 120;
    
    // Setup diagnostic visualizer
    if (analyser) {
        diagnosticAnalyser = analyser;
        drawDiagnosticWave();
    }
    
    // Update device info
    updateDiagnosticDeviceInfo();
    
    // Start periodic update
    diagnosticInterval = setInterval(updateDiagnosticPanel, 100);
}

function stopDiagnostic() {
    if (diagnosticInterval) {
        clearInterval(diagnosticInterval);
        diagnosticInterval = null;
    }
}

function drawDiagnosticWave() {
    if (!isListening || !diagnosticAnalyser || !diagnosticCanvas || !diagnosticCtx) {
        requestAnimationFrame(drawDiagnosticWave);
        return;
    }
    
    requestAnimationFrame(drawDiagnosticWave);
    
    const bufferLength = diagnosticAnalyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);
    diagnosticAnalyser.getByteFrequencyData(dataArray);
    
    // Calculate average volume
    const average = dataArray.reduce((a, b) => a + b, 0) / bufferLength;
    diagnosticData.audioLevel = Math.round((average / 255) * 100);
    
    // Draw waveform
    diagnosticCtx.fillStyle = '#1e293b';
    diagnosticCtx.fillRect(0, 0, diagnosticCanvas.width, diagnosticCanvas.height);
    
    const barWidth = (diagnosticCanvas.width / bufferLength) * 2.5;
    let x = 0;
    
    for (let i = 0; i < bufferLength; i++) {
        const barHeight = (dataArray[i] / 255) * diagnosticCanvas.height;
        const hue = 120 - (dataArray[i] / 255) * 120; // Green to Red
        diagnosticCtx.fillStyle = `hsl(${hue}, 70%, 50%)`;
        diagnosticCtx.fillRect(x, diagnosticCanvas.height - barHeight, barWidth, barHeight);
        x += barWidth + 1;
    }
}

function updateDiagnosticPanel() {
    const now = Date.now();
    
    // Update volume
    const volEl = document.getElementById('diag-volume');
    if (volEl) volEl.textContent = diagnosticData.audioLevel + '%';
    
    // Update recognition state
    const stateEl = document.getElementById('diag-recognition-state');
    if (stateEl) {
        if (isListening) {
            stateEl.textContent = 'ACTIVE';
            stateEl.className = 'status-badge status-active';
        } else {
            stateEl.textContent = 'STOPPED';
            stateEl.className = 'status-badge status-stopped';
        }
    }
    
    // Update last start time
    const startEl = document.getElementById('diag-last-start');
    if (startEl && diagnosticData.lastStartTime) {
        startEl.textContent = new Date(diagnosticData.lastStartTime).toLocaleTimeString();
    }
    
    // Update last result time
    const resultEl = document.getElementById('diag-last-result');
    if (resultEl && diagnosticData.lastResultTime) {
        resultEl.textContent = new Date(diagnosticData.lastResultTime).toLocaleTimeString();
    }
    
    // Update time since last result
    const timeSinceEl = document.getElementById('diag-time-since-result');
    if (timeSinceEl && diagnosticData.lastResultTime) {
        const seconds = Math.floor((now - diagnosticData.lastResultTime) / 1000);
        timeSinceEl.textContent = seconds + ' 秒前';
        
        // Color code based on time
        if (seconds > 10) {
            timeSinceEl.className = 'text-sm font-bold text-red-600';
        } else if (seconds > 5) {
            timeSinceEl.className = 'text-sm font-bold text-amber-600';
        } else {
            timeSinceEl.className = 'text-sm font-bold text-green-600';
        }
    }
    
    // Update error
    const errorEl = document.getElementById('diag-last-error');
    if (errorEl) {
        if (diagnosticData.lastErrorMessage) {
            errorEl.textContent = diagnosticData.lastErrorMessage;
        } else {
            errorEl.textContent = '無錯誤';
            errorEl.className = 'text-sm text-green-500 font-mono';
        }
    }
    
    // Update silence countdown
    if (isListening && diagnosticData.lastResultTime) {
        const elapsed = (now - diagnosticData.lastResultTime) / 1000;
        const remaining = Math.max(0, 15 - elapsed);
        
        const countdownEl = document.getElementById('diag-silence-countdown');
        if (countdownEl) {
            countdownEl.textContent = remaining.toFixed(1) + ' 秒';
            
            if (remaining < 3) {
                countdownEl.className = 'text-lg font-bold text-red-600 animate-pulse';
            } else if (remaining < 8) {
                countdownEl.className = 'text-lg font-bold text-amber-600';
            } else {
                countdownEl.className = 'text-lg font-bold text-green-600';
            }
        }
        
        const barEl = document.getElementById('diag-silence-bar');
        if (barEl) {
            const percentage = (remaining / 15) * 100;
            barEl.style.width = percentage + '%';
        }
    } else {
        const countdownEl = document.getElementById('diag-silence-countdown');
        if (countdownEl) countdownEl.textContent = '--';
        
        const barEl = document.getElementById('diag-silence-bar');
        if (barEl) barEl.style.width = '100%';
    }
}

function updateDiagnosticDeviceInfo() {
    const deviceEl = document.getElementById('diag-device-name');
    const langEl = document.getElementById('diag-language');
    
    if (deviceEl && audioSelect) {
        const selectedOption = audioSelect.options[audioSelect.selectedIndex];
        deviceEl.textContent = selectedOption ? selectedOption.text : '未知';
    }
    
    if (langEl) {
        langEl.textContent = currentLang;
    }
}

function copyDiagnosticReport() {
    const report = `
=== Vibe Voice 音訊診斷報告 ===
時間: ${new Date().toLocaleString()}

【麥克風狀態】
音量強度: ${diagnosticData.audioLevel}%
裝置: ${audioSelect ? audioSelect.options[audioSelect.selectedIndex].text : '未知'}

【語音識別狀態】
Recognition 狀態: ${isListening ? 'ACTIVE' : 'STOPPED'}
上次啟動: ${diagnosticData.lastStartTime ? new Date(diagnosticData.lastStartTime).toLocaleString() : '未啟動'}
上次接收結果: ${diagnosticData.lastResultTime ? new Date(diagnosticData.lastResultTime).toLocaleString() : '無資料'}
距離上次結果: ${diagnosticData.lastResultTime ? Math.floor((Date.now() - diagnosticData.lastResultTime) / 1000) + ' 秒' : 'N/A'}
最後錯誤: ${diagnosticData.lastErrorMessage || '無錯誤'}

【靜音偵測】
超時設定: 15 秒
剩餘時間: ${diagnosticData.lastResultTime ? Math.max(0, 15 - (Date.now() - diagnosticData.lastResultTime) / 1000).toFixed(1) + ' 秒' : 'N/A'}

【系統資訊】
語言設定: ${currentLang}
模式: ${currentMode}
AI 翻譯: ${isAIEnabled ? '開啟' : '關閉'}
    `.trim();
    
    copyTextToClipboard(report);
    showToast('診斷報告已複製', 'success');
}
```

### 6. 修改 initRecognition() 函式以追蹤事件（約第 1024 行）

在 `recognition.onstart` 事件中新增：
```javascript
recognition.onstart = function() {
    logDebug("Recognition started", "sys");
    diagnosticData.lastStartTime = Date.now();
    diagnosticData.recognitionState = 'active';
    // ... 原有程式碼
};
```

在 `recognition.onresult` 事件中新增：
```javascript
recognition.onresult = function(event) {
    diagnosticData.lastResultTime = Date.now();
    lastResultTime = Date.now(); // 更新現有變數
    // ... 原有程式碼
};
```

在 `recognition.onerror` 事件中新增：
```javascript
recognition.onerror = function(event) {
    diagnosticData.lastErrorTime = Date.now();
    diagnosticData.lastErrorMessage = event.error + ' - ' + (event.message || '');
    logDebug(`Recognition error: ${event.error}`, "error");
    // ... 原有程式碼
};
```

在 `recognition.onend` 事件中新增：
```javascript
recognition.onend = function() {
    diagnosticData.recognitionState = 'stopped';
    // ... 原有程式碼
};
```

### 7. 修改 toggleModal 函式以支援 diagnostic-modal（第 531 行）

確保 toggleModal 函式可以正確處理 diagnostic-modal。

## 使用方式

1. 開啟應用程式
2. 點擊右上角的 🩺 (聽診器) �示開啟診斷視窗
3. 開始錄音
4. 即時觀察：
   - **波形圖**：確認麥克風是否有訊號
   - **onresult 時間**：確認 Web Speech API 是否正常回傳結果
   - **靜音倒數**：觀察是否快要觸發自動停止
   - **錯誤訊息**：檢查是否有 API 錯誤

## 診斷判斷邏輯

| 症狀 | 可能原因 | 建議處理 |
|------|---------|---------|
| 有波形 + 無 onresult | Web Speech API 未識別 | 檢查語言設定、嘗試重新授權麥克風 |
| 無波形 + 無 onresult | 麥克風未啟動 | 檢查瀏覽器權限、裝置設定 |
| 有 onresult + 倒數歸零 | 靜音偵測誤判 | 調整 SILENCE_TIMEOUT 數值 |
| 有錯誤訊息 | API 層級問題 | 根據錯誤代碼處理 |

## 版本資訊

- 實作版本: v1.9.2
- 新增功能: 音訊診斷視窗
- 修改檔案: index.html

---

實作完成後，請將版本號從 v1.9.0 更新為 v1.9.2 (第 6 行 title)