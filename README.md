<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Терминал Сбера | FaceID</title>
  <style>
    body {
      margin: 0;
      font-family: 'Segoe UI', sans-serif;
      background: linear-gradient(to bottom, #000, #0a2e0a);
      color: #00ff88;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      overflow: hidden;
    }
    .terminal {
      width: 400px;
      height: 600px;
      background: #000;
      border: 2px solid #00ff88;
      border-radius: 16px;
      padding: 20px;
      box-shadow: 0 0 20px rgba(0, 255, 136, 0.3);
      position: relative;
    }
    .header {
      text-align: center;
      font-size: 20px;
      font-weight: bold;
      margin-bottom: 20px;
      color: #00ff88;
    }
    .screen {
      background: #001a0a;
      border: 1px solid #00ff88;
      border-radius: 10px;
      height: 400px;
      padding: 15px;
      margin-bottom: 20px;
      overflow: hidden;
      position: relative;
    }
    .status {
      font-size: 16px;
      margin: 10px 0;
      text-align: center;
      min-height: 24px;
    }
    .input-amount {
      text-align: center;
      margin: 20px 0;
    }
    .input-amount input {
      background: #001a0a;
      border: 1px solid #00ff88;
      color: #00ff88;
      padding: 10px;
      width: 120px;
      text-align: center;
      font-size: 18px;
      border-radius: 6px;
    }
    #videoContainer {
      display: none;
      text-align: center;
      margin-top: 10px;
    }
    #video {
      width: 180px;
      height: 140px;
      border: 1px solid #00ff88;
      border-radius: 8px;
      background: #000;
      transform: scaleX(-1); /* Зеркало, как в селфи-камере */
    }
    .scan-overlay {
      position: absolute;
      top: 0;
      left: 0;
      width: 180px;
      height: 140px;
      margin: 0 auto;
      right: 0;
      background: 
        radial-gradient(circle at 50% 50%, transparent 40%, rgba(0, 255, 136, 0.1) 60%, rgba(0, 255, 136, 0.3) 70%, transparent 75%);
      border-radius: 8px;
      pointer-events: none;
      animation: scanPulse 2s infinite;
    }
    @keyframes scanPulse {
      0% { opacity: 0.5; transform: scale(1); }
      50% { opacity: 0.8; transform: scale(1.05); }
      100% { opacity: 0.5; transform: scale(1); }
    }
    .amount {
      text-align: center;
      font-size: 24px;
      font-weight: bold;
      color: #00ff88;
      margin-top: 10px;
    }
    .key-hint {
      position: absolute;
      bottom: 10px;
      left: 10px;
      font-size: 12px;
      color: #00aa55;
    }
    .success { color: #00ff88; }
    .error { color: #ff3300; }
    .warning { color: #ffcc00; }
  </style>
</head>
<body>
  <div class="terminal">
    <div class="header">Сбербанк</div>
    <div class="screen">
      <div id="status" class="status">Введите сумму для оплаты</div>
      <div class="input-amount">
        <input type="number" id="sumInput" placeholder="Сумма" min="1" max="50000" step="1"/>
      </div>
      <div id="videoContainer">
        <div style="position: relative; display: inline-block;">
          <video id="video" autoplay playsinline></video>
          <div class="scan-overlay"></div>
        </div>
      </div>
      <div id="amount" class="amount" style="display: none;"></div>
    </div>
    <div class="key-hint">F1 — FaceID, F2 — NFC, Enter — ввести сумму, F3 — оплатить</div>
  </div>

  <script>
    // Звуки
    const sounds = {
      scan: new Audio("https://www.soundjay.com/button/sounds/button-09.mp3"),
      success: new Audio("https://www.soundjay.com/misc/sounds/bell-ringing-05.mp3"),
      error: new Audio("https://www.soundjay.com/button/sounds/button-10.mp3"),
      faceid: new Audio("https://www.soundjay.com/misc/sounds/camera-shutter-1.mp3"),
      keypress: new Audio("https://www.soundjay.com/misc/sounds/typewriter-key-01.mp3")
    };

    Object.values(sounds).forEach(sound => {
      sound.preload = "auto";
    });

    // Элементы
    const status = document.getElementById("status");
    const sumInput = document.getElementById("sumInput");
    const video = document.getElementById("video");
    const videoContainer = document.getElementById("videoContainer");
    const amount = document.getElementById("amount");

    // Данные
    const cardNumber = "2200 7012 2258 8524";
    let balance = 15000;
    let transferAmount = 0;
    let stream = null;

    // Установка статуса
    function setStatus(text, className = "") {
      status.textContent = text;
      status.className = "status " + className;
      sounds.keypress.play().catch(() => {});
    }

    // Очистка камеры
    async function stopCamera() {
      if (stream) {
        stream.getTracks().forEach(track => track.stop());
        stream = null;
      }
      video.srcObject = null;
      videoContainer.style.display = "none";
    }

    // Запуск камеры
    async function startCamera() {
      if (transferAmount === 0) {
        setStatus("Сначала введите сумму", "error");
        sounds.error.play().catch(() => {});
        return;
      }

      try {
        setStatus("Запрос доступа к камере...");
        stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: "user" } });
        video.srcObject = stream;
        videoContainer.style.display = "block";
        setStatus("Сканирование лица...");
        sounds.faceid.play().catch(() => {});

        setTimeout(() => {
          setStatus("Лицо распознано");
          setTimeout(() => {
            document.addEventListener("keydown", handlePaymentKey, { once: true });
            setStatus("Готово. Нажмите F3 для оплаты");
          }, 800);
        }, 2500);

      } catch (err) {
        setStatus("Ошибка камеры: отклонено", "error");
        console.error("Доступ к камере отклонён:", err);
        sounds.error.play().catch(() => {});
      }
    }

    // Ввод суммы
    sumInput.addEventListener("keypress", (e) => {
      if (e.key === "Enter") {
        const value = parseFloat(sumInput.value);
        if (!value || value <= 0) {
          setStatus("Ошибка: введите сумму", "error");
          sounds.error.play().catch(() => {});
          return;
        }
        if (value > 50000) {
          setStatus("Макс. сумма — 50 000 ₽", "warning");
          sounds.error.play().catch(() => {});
          return;
        }
        transferAmount = value;
        amount.textContent = `Сумма: ${transferAmount} ₽`;
        amount.style.display = "block";
        setStatus("Сумма введена. Нажмите F1 (FaceID) или F2 (NFC)");
        sumInput.disabled = true;
      }
    });

    // NFC (без камеры)
    function startNFC() {
      if (transferAmount === 0) {
        setStatus("Сначала введите сумму", "error");
        sounds.error.play().catch(() => {});
        return;
      }
      stopCamera();
      setStatus("Поднесите карту...");
      sounds.scan.play().catch(() => {});

      setTimeout(() => {
        setStatus("Карта обнаружена");
        setTimeout(() => {
          document.addEventListener("keydown", handlePaymentKey, { once: true });
          setStatus("Готово. Нажмите F3 для оплаты");
        }, 800);
      }, 1500);
    }

    // Оплата
    function processPayment() {
      if (transferAmount === 0) return;

      if (balance < transferAmount) {
        setStatus("Ошибка: недостаточно средств", "error");
        sounds.error.play().catch(() => {});
        return;
      }

      balance -= transferAmount;
      setStatus(`✅ Оплата ${transferAmount} ₽ прошла!\nОстаток: ${balance} ₽`, "success");
      sounds.success.play().catch(() => {});

      console.log(`Переведено ${transferAmount} ₽ на карту ${cardNumber}`);
      setTimeout(() => {
        resetTerminal();
      }, 4000);
    }

    // Сброс
    function resetTerminal() {
      stopCamera();
      amount.style.display = "none";
      sumInput.disabled = false;
      sumInput.value = "";
      transferAmount = 0;
      setStatus("Введите сумму для оплаты");
    }

    // Горячие клавиши
    document.addEventListener("keydown", (e) => {
      if (e.key === "F1" || e.keyCode === 112) {
        e.preventDefault();
        startCamera();
      } else if (e.key === "F2" || e.keyCode === 113) {
        e.preventDefault();
        startNFC();
      } else if (e.key === "F3" || e.keyCode === 114) {
        e.preventDefault();
        processPayment();
      }
    });

    // Инициализация
    setStatus("Введите сумму и нажмите Enter");

    // Очистка при выходе
    window.addEventListener("beforeunload", stopCamera);
  </script>
</body>
</html>
