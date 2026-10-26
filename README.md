
Model 6:52 PM
ThinkingThoughts
Expand to view model thoughts

chevron_right
بكل شفافية ووضوح، بيئة المحادثة الحالية هي واجهة ذكاء اصطناعي نصية لا تحتوي على خادم تشغيل سحابي أو نظام طرفية (Terminal/Build Sandbox) لتشغيل خادم Node.js وتوليد رابط معاينة حيّ داخل هذه النافذة مباشرة. ولأنك طلبت عدم الادعاء بتشغيل خادم وهمي أو نشر وهمي، فلن أزعم أن الموقع منشور حالياً على الإنترنت.
ومع ذلك، قمت ببناء مشروع برمجي متكامل، حقيقي، وخالٍ من الأكواد الوهمية، مصمم خصيصاً ليتم نشره في دقائق معدودة مجاناً مباشرة من متصفح هاتف أندرويد.
ملفات المشروع البرمجي المتكامل
1. ملف التبعيات والإعدادات (package.json)
code
JSON
{
  "name": "smart-transcriber",
  "version": "1.0.0",
  "description": "تطبيق المفرغ الذكي لتحويل الصوت إلى نص",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "form-data": "^4.0.0",
    "multer": "^1.4.5-lts.1",
    "node-fetch": "^2.7.0"
  }
}
2. الخادم الخلفي (server.js)
يعتمد هذا الخادم على خدمة Groq Cloud التي تشغّل نموذج Whisper-Large-v3، وهو أقوى نموذج لتفريغ اللهجات العربية (بما فيها اللهجة العراقية)، ومجاني وسريع جداً. يقوم الخادم باستلام الملف بأمان، وإرساله إلى محرك التفريغ، وتنظيف الملفات المؤقتة تلقائياً فور الانتهاء.
code
JavaScript
const express = require('express');
const multer = require('multer');
const cors = require('cors');
const fs = require('fs');
const path = require('path');
const fetch = require('node-fetch');
const FormData = require('form-data');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

// مجلد التحميل المؤقت
const uploadDir = path.join(__dirname, 'uploads');
if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir, { recursive: true });
}

// إعداد التخزين الآمن
const storage = multer.diskStorage({
    destination: (req, file, cb) => cb(null, uploadDir),
    filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
        cb(null, uniqueSuffix + path.extname(file.originalname));
    }
});

const upload = multer({
    storage: storage,
    limits: { fileSize: 50 * 1024 * 1024 } // حد أقصى 50 ميغابايت للملف
});

app.use(cors());
app.use(express.static('public'));

// مسار تفريغ الصوت
app.post('/api/transcribe', upload.single('audio'), async (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: 'لم يتم استلام أي ملف صوتي.' });
    }

    const apiKey = process.env.GROQ_API_KEY;
    if (!apiKey) {
        // حذف الملف المؤقت في حال عدم وجود المفتاح
        if (fs.existsSync(req.file.path)) fs.unlinkSync(req.file.path);
        return res.status(500).json({ 
            error: 'مفتاح GROQ_API_KEY غير مهيأ على الخادم. يرجى إضافته في إعدادات البيئة.' 
        });
    }

    try {
        const formData = new FormData();
        formData.append('file', fs.createReadStream(req.file.path));
        formData.append('model', 'whisper-large-v3');
        formData.append('language', 'ar');
        formData.append('response_format', 'verbose_json');

        const response = await fetch('https://api.groq.com/openai/v1/audio/transcriptions', {
            method: 'POST',
            headers: {
                'Authorization': `Bearer ${apiKey}`,
                ...formData.getHeaders()
            },
            body: formData
        });

        // حذف الملف الصوتي المؤقت فوراً من الخادم لحفظ الخصوصية والمساحة
        if (fs.existsSync(req.file.path)) {
            fs.unlinkSync(req.file.path);
        }

        if (!response.ok) {
            const errData = await response.json().catch(() => ({}));
            return res.status(response.status).json({ 
                error: errData.error?.message || 'فشلت معالجة الصوت في محرك التفريغ.' 
            });
        }

        const data = await response.json();
        return res.json({ 
            text: data.text,
            duration: data.duration || null
        });

    } catch (error) {
        if (fs.existsSync(req.file.path)) fs.unlinkSync(req.file.path);
        return res.status(500).json({ error: 'حدث خطأ غير متوقع أثناء معالجة الملف: ' + error.message });
    }
});

app.listen(PORT, () => {
    console.log(`Smart Transcriber server running on port ${PORT}`);
});
3. واجهة المستخدم (public/index.html)
تدعم اتجاه RTL، ومتوافقة بالكامل مع شاشات الهواتف ومتصفحات أندرويد:
code
Html
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>المفرّغ الذكي | Smart Transcriber</title>
    <link rel="stylesheet" href="style.css">
    <!-- مكتبة html2pdf لتوليد ملفات PDF باللغة العربية مع الحفاظ على ترابط الحروف السليم -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
</head>
<body>
    <div class="container">
        <header class="app-header">
            <div class="logo-icon">🎙️</div>
            <h1>المفرّغ الذكي</h1>
            <p class="subtitle">تحويل التسجيلات الصوتية العربية والعراقية إلى نص بدقة عالية</p>
        </header>

        <section class="card upload-section">
            <input type="file" id="audioInput" accept=".mp3,.wav,.m4a,audio/*" style="display: none;">
            <button class="btn-primary" id="btnSelectFile" onclick="document.getElementById('audioInput').click()">
                <span>📁</span> اختيار ملف صوتي
            </button>
            <p class="formats-hint">الصيغ المدعومة: MP3, WAV, M4A</p>

            <div id="fileInfoCard" class="file-info-card" style="display: none;">
                <div class="file-details">
                    <p><strong>اسم الملف:</strong> <span id="fileName">-</span></p>
                    <p><strong>الحجم:</strong> <span id="fileSize">-</span></p>
                    <p><strong>المدة التقديرية:</strong> <span id="fileDuration">جارٍ القياس...</span></p>
                </div>
                <button class="btn-action" id="btnStartTranscribe">بدء التفريغ الآن</button>
            </div>
        </section>

        <div id="statusBox" class="status-box" style="display: none;"></div>

        <section class="card result-section" id="resultCard" style="display: none;">
            <div class="result-header">
                <h3>النص المفرّغ (قابل للتعديل):</h3>
                <span id="wordCountBadge" class="badge">0 كلمة</span>
            </div>
            
            <div id="pdfContent">
                <textarea id="transcriptionText" rows="10" placeholder="النص سيظهر هنا..."></textarea>
            </div>

            <div class="tools-grid">
                <button class="tool-btn" id="btnCopy">📋 نسخ النص</button>
                <button class="tool-btn" id="btnDownloadTxt">📄 تنزيل TXT</button>
                <button class="tool-btn" id="btnDownloadPdf">📑 تنزيل PDF</button>
                <button class="tool-btn" id="btnShareText">💬 مشاركة النص</button>
                <button class="tool-btn" id="btnSharePdf">📤 مشاركة ملف PDF</button>
                <button class="tool-btn" id="btnShareAudio">🎵 مشاركة الصوت</button>
                <button class="tool-btn" id="btnShareApp">🌐 مشاركة التطبيق</button>
            </div>
        </section>
    </div>

    <script src="app.js"></script>
</body>
</html>
4. التنسيقات المتجاوبة (public/style.css)
code
CSS
:root {
    --primary: #0284c7;
    --primary-hover: #0369a1;
    --bg-page: #f8fafc;
    --card-bg: #ffffff;
    --text-main: #0f172a;
    --text-muted: #64748b;
    --border: #e2e8f0;
    --success: #10b981;
    --error: #ef4444;
}

* {
    box-sizing: border-box;
    font-family: system-ui, -apple-system, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
    margin: 0;
    padding: 0;
}

body {
    background-color: var(--bg-page);
    color: var(--text-main);
    direction: rtl;
    padding: 16px;
    display: flex;
    justify-content: center;
}

.container {
    width: 100%;
    max-width: 600px;
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.app-header {
    text-align: center;
    padding: 12px 0;
}

.logo-icon {
    font-size: 42px;
    margin-bottom: 6px;
}

.app-header h1 {
    font-size: 24px;
    color: var(--primary);
    font-weight: 800;
}

.subtitle {
    font-size: 13px;
    color: var(--text-muted);
    margin-top: 4px;
}

.card {
    background: var(--card-bg);
    border-radius: 14px;
    padding: 18px;
    border: 1px solid var(--border);
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05);
}

.upload-section {
    text-align: center;
}

.btn-primary {
    width: 100%;
    background-color: var(--primary);
    color: white;
    border: none;
    padding: 14px;
    border-radius: 10px;
    font-size: 16px;
    font-weight: bold;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
}

.formats-hint {
    font-size: 12px;
    color: var(--text-muted);
    margin-top: 8px;
}

.file-info-card {
    margin-top: 14px;
    padding: 12px;
    background: #f1f5f9;
    border-radius: 8px;
    text-align: right;
    font-size: 14px;
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.btn-action {
    margin-top: 10px;
    background-color: var(--success);
    color: white;
    border: none;
    padding: 12px;
    border-radius: 8px;
    font-size: 15px;
    font-weight: bold;
    cursor: pointer;
}

.status-box {
    padding: 12px;
    border-radius: 8px;
    font-size: 14px;
    text-align: center;
    font-weight: 500;
}

.status-loading { background: #e0f2fe; color: #0369a1; }
.status-error { background: #fee2e2; color: #991b1b; }
.status-success { background: #dcfce7; color: #166534; }

.result-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 8px;
}

.badge {
    background: #f1f5f9;
    padding: 4px 8px;
    border-radius: 6px;
    font-size: 12px;
    color: var(--text-muted);
}

textarea {
    width: 100%;
    padding: 12px;
    border: 1px solid var(--border);
    border-radius: 8px;
    font-size: 15px;
    line-height: 1.6;
    direction: rtl;
    resize: vertical;
    outline: none;
}

.tools-grid {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 8px;
    margin-top: 14px;
}

.tool-btn {
    background: #f8fafc;
    border: 1px solid var(--border);
    padding: 10px;
    border-radius: 8px;
    font-size: 13px;
    font-weight: 600;
    cursor: pointer;
    text-align: center;
}

.tool-btn:active {
    background: #e2e8f0;
}
5. منطق المتصفح التفاعلي (public/app.js)
code
JavaScript
let currentFile = null;

const audioInput = document.getElementById('audioInput');
const fileInfoCard = document.getElementById('fileInfoCard');
const fileName = document.getElementById('fileName');
const fileSize = document.getElementById('fileSize');
const fileDuration = document.getElementById('fileDuration');
const btnStartTranscribe = document.getElementById('btnStartTranscribe');
const statusBox = document.getElementById('statusBox');
const resultCard = document.getElementById('resultCard');
const transcriptionText = document.getElementById('transcriptionText');
const wordCountBadge = document.getElementById('wordCountBadge');

// عند اختيار ملف صوتي
audioInput.addEventListener('change', function(e) {
    if (this.files && this.files[0]) {
        currentFile = this.files[0];
        fileName.textContent = currentFile.name;
        fileSize.textContent = (currentFile.size / (1024 * 1024)).toFixed(2) + ' ميغابايت';
        fileInfoCard.style.display = 'flex';
        
        // حساب مدة المقطع الصوتي
        const audio = new Audio();
        audio.src = URL.createObjectURL(currentFile);
        audio.onloadedmetadata = function() {
            const minutes = Math.floor(audio.duration / 60);
            const seconds = Math.floor(audio.duration % 60);
            fileDuration.textContent = `${minutes} دقيقة و ${seconds} ثانية`;
        };
        audio.onerror = function() {
            fileDuration.textContent = 'تعذر قراءة المدة تلقائياً';
        };
    }
});

// بدء عملية التفريغ
btnStartTranscribe.addEventListener('click', async function() {
    if (!currentFile) return;

    statusBox.style.display = 'block';
    statusBox.className = 'status-box status-loading';
    statusBox.textContent = '⏳ جارٍ رفع الصوت وتحليله عبر محرك الذكاء الاصطناعي... يرجى الانتظار.';
    btnStartTranscribe.disabled = true;

    const formData = new FormData();
    formData.append('audio', currentFile);

    try {
        const response = await fetch('/api/transcribe', {
            method: 'POST',
            body: formData
        });

        const data = await response.json();

        if (!response.ok) {
            throw new Error(data.error || 'حدث خطأ غير معروف في الخادم');
        }

        transcriptionText.value = data.text;
        updateWordCount();
        resultCard.style.display = 'block';
        statusBox.className = 'status-box status-success';
        statusBox.textContent = '✅ اكتمل التفريغ بنجاح!';
        
        resultCard.scrollIntoView({ behavior: 'smooth' });

    } catch (err) {
        statusBox.className = 'status-box status-error';
        statusBox.textContent = '❌ خطأ: ' + err.message;
    } finally {
        btnStartTranscribe.disabled = false;
    }
});

// تحديث عدد الكلمات
transcriptionText.addEventListener('input', updateWordCount);

function updateWordCount() {
    const text = transcriptionText.value.trim();
    const count = text ? text.split(/\s+/).length : 0;
    wordCountBadge.textContent = count + ' كلمة';
}

// 1. نسخ النص
document.getElementById('btnCopy').addEventListener('click', () => {
    navigator.clipboard.writeText(transcriptionText.value);
    alert('تم نسخ النص إلى الحافظة.');
});

// 2. تنزيل ملف TXT مع دعم الترميز العربي UTF-8
document.getElementById('btnDownloadTxt').addEventListener('click', () => {
    const blob = new Blob(['\ufeff' + transcriptionText.value], { type: 'text/plain;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `تفريغ_${Date.now()}.txt`;
    a.click();
    URL.revokeObjectURL(url);
});

// 3. تنزيل ملف PDF مع ضمان ترابط الحروف العربية
document.getElementById('btnDownloadPdf').addEventListener('click', () => {
    const element = document.getElementById('transcriptionText');
    const opt = {
        margin: 15,
        filename: `تفريغ_${Date.now()}.pdf`,
        image: { type: 'jpeg', quality: 0.98 },
        html2canvas: { scale: 2 },
        jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };
    html2pdf().set(opt).from(element).save();
});

// 4. مشاركة النص عبر Web Share API
document.getElementById('btnShareText').addEventListener('click', async () => {
    if (navigator.share) {
        try {
            await navigator.share({
                title: 'النص المفرّغ - المفرّغ الذكي',
                text: transcriptionText.value
            });
        } catch (e) {}
    } else {
        alert('ميزة المشاركة المباشرة غير مدعومة في متصفحك، تم نسخ النص بدلاً من ذلك.');
        navigator.clipboard.writeText(transcriptionText.value);
    }
});

// 5. مشاركة ملف PDF الفعلي عبر قائمة المشاركة الأصلية في أندرويد
document.getElementById('btnSharePdf').addEventListener('click', async () => {
    const element = document.getElementById('transcriptionText');
    const opt = {
        margin: 15,
        html2canvas: { scale: 2 },
        jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };

    try {
        const pdfWorker = html2pdf().set(opt).from(element);
        const pdfBlob = await pdfWorker.outputPdf('blob');
        const pdfFile = new File([pdfBlob], 'transcription.pdf', { type: 'application/pdf' });

        if (navigator.canShare && navigator.canShare({ files: [pdfFile] })) {
            await navigator.share({
                files: [pdfFile],
                title: 'تفريغ صوتي PDF',
                text: 'إليك المستند المفرغ بصيغة PDF'
            });
        } else {
            alert('متصفحك لا يدعم مشاركة الملفات مباشرة، سيتم تنزيل الـ PDF بدلاً من ذلك.');
            pdfWorker.save();
        }
    } catch (err) {
        alert('تعذرت مشاركة الملف: ' + err.message);
    }
});

// 6. مشاركة الملف الصوتي الأصلي
document.getElementById('btnShareAudio').addEventListener('click', async () => {
    if (!currentFile) {
        alert('يرجى اختيار ملف صوتي أولاً');
        return;
    }
    if (navigator.canShare && navigator.canShare({ files: [currentFile] })) {
        try {
            await navigator.share({
                files: [currentFile],
                title: currentFile.name,
                text: 'الملف الصوتي الأصلي'
            });
        } catch (e) {}
    } else {
        alert('متصفحك لا يدعم مشاركة الملف الصوتي مباشرة.');
    }
});

// 7. مشاركة رابط التطبيق
document.getElementById('btnShareApp').addEventListener('click', async () => {
    if (navigator.share) {
        await navigator.share({
            title: 'تطبيق المفرّغ الذكي',
            text: 'تطبيق مجاني وسريع لتفريغ التسجيلات الصوتية العربية والعراقية إلى نص',
            url: window.location.href
        });
    } else {
        navigator.clipboard.writeText(window.location.href);
        alert('تم نسخ رابط التطبيق إلى الحافظة.');
    }
});
الفرق بين المعاينة المحلية والرابط العام المنشور
المعاينة المحلية (Localhost / Preview): تعني أن التطبيق يعمل فقط داخل معالج الهاتف أو الحاسوب الشخصي على عنوان داخلي مثل http://localhost:3000. لا يستطيع أي شخص آخر فتحه، وتتوقف بمجرد إغلاق نافذة البرنامج.
الرابط العام المنشور (Live Public URL): هو موقع حقيقي مستضاف على خادم متصل بالإنترنت على مدار الساعة، يحمل عنواناً مثل https://smart-transcriber.onrender.com، يعمل بشهادة أمان SSL (HTTPS)، ويمكن لأي شخص في العالم فتحه واستخدامه على هاتفه.
طريقة النشر المجاني خطوة بخطوة من هاتف أندرويد (بدون خبرة برمجية)
أنسب وأبسط منصة سحابية لتشغيل هذا التطبيق مجاناً وبدون بطاقة بنكية هي منصة Glitch أو منصة Render:
الحصول على مفتاح Groq API المجاني (دقيقة واحدة):
افتح متصفح هاتفك وادخل إلى console.groq.com.
سجّل دخولك بحساب Google الخاص بك.
اضغط على القائمة ثم اختر API Keys، ثم Create API Key.
انسخ المفتاح الذي يظهر لك واحفظه عندك.
رفع التطبيق وتشغيله بنقرة واحدة عبر Glitch (من الهاتف مباشرة):
افتح glitch.com من هاتف أندرويد وسجّل دخولك بحساب Google.
اضغط على New Project واختر glitch-hello-node.
ستجد قائمة ملفات على اليسار: احذف الملفات الموجودة واستبدلها بالملفات الـ 5 المكتوبة أعلاه (server.js, package.json, وداخل مجلد public ضع index.html, style.css, app.js).
اضغط على ملف باسم .env أو زر Add Variable، واكتب:
الاسم: GROQ_API_KEY
القيمة: الصق المفتاح الذي نسخته من الخطوة (1).
سيعطيك الموقع فوراً رابطاً عاماً بصيغة # smart-transcriber