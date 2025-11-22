<!DOCTYPE html>
<html lang="pl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Konwerter AI Math do Worda</title>
    <script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
    <script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: #f4f4f9;
            color: #333;
            margin: 0;
            padding: 20px;
            display: flex;
            flex-direction: column;
            align-items: center;
            min-height: 100vh;
        }

        .container {
            background: white;
            padding: 30px;
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
            width: 100%;
            max-width: 900px;
        }

        h1 {
            text-align: center;
            color: #2c3e50;
            margin-bottom: 10px;
        }

        p.desc {
            text-align: center;
            color: #666;
            margin-bottom: 25px;
        }

        .editor-box {
            margin-bottom: 20px;
        }

        label {
            display: block;
            font-weight: bold;
            margin-bottom: 8px;
            color: #2c3e50;
        }

        textarea {
            width: 100%;
            height: 150px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 8px;
            font-family: monospace;
            resize: vertical;
            box-sizing: border-box;
        }

        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 12px 24px;
            font-size: 16px;
            border-radius: 6px;
            cursor: pointer;
            transition: background-color 0.2s;
            display: block;
            margin: 0 auto 20px auto;
        }

        button:hover {
            background-color: #0056b3;
        }

        button.copy-btn {
            background-color: #28a745;
        }

        button.copy-btn:hover {
            background-color: #218838;
        }

        #preview {
            padding: 20px;
            border: 1px solid #ccc;
            border-radius: 8px;
            background-color: #fff;
            min-height: 100px;
            margin-bottom: 20px;
            /* To jest kluczowe dla Worda - czcionka szeryfowa przypomina domyślny styl Worda */
            font-family: 'Times New Roman', Times, serif;
            line-height: 1.5;
        }

        /* Ukrywamy komunikaty MathJax podczas przetwarzania */
        mjx-container {
            margin: 5px 0 !important;
        }

        .status {
            text-align: center;
            margin-top: 10px;
            font-weight: bold;
            height: 20px;
            color: #28a745;
        }
    </style>
</head>
<body>

<div class="container">
    <h1>Konwerter AI Math → Word</h1>
    <p class="desc">Poproś AI o wygenerowanie obliczeń w formacie LaTeX. Wklej odpowiedź z AI (z kodem LaTeX), przekonwertuj, a następnie skopiuj gotowy sformatowany tekst do Worda.</p>

    <div class="editor-box">
        <label for="input-text">1. Wklej tekst z AI tutaj:</label>
        <textarea id="input-text" placeholder="Np. Oblicz całkę: $$\int_{0}^{1} x^2 dx$$ ..."></textarea>
    </div>

    <button onclick="renderText()">2. Przetwórz na format Worda</button>

    <div class="editor-box">
        <label>3. Podgląd (Gotowe do skopiowania):</label>
        <div id="preview"></div>
    </div>

    <button class="copy-btn" onclick="copyToWord()">4. Skopiuj do schowka (Specjalny format)</button>
    <div class="status" id="status-msg"></div>
</div>

<script>
    // Konfiguracja MathJax, aby wymusić wyjście SVG/CommonHTML, 
    // ale my użyjemy tricku z MML (MathML) dla Worda.
    window.MathJax = {
        tex: {
            inlineMath: [['$', '$'], ['\\(', '\\)']],
            displayMath: [['$$', '$$'], ['\\[', '\\]']],
            processEscapes: true
        },
        options: {
            // Wyłączamy menu kontekstowe MathJax, żeby nie przeszkadzało przy kopiowaniu
            enableMenu: false
        },
        startup: {
            // Ważne: chcemy, aby MathJax przetworzył stronę
            pageReady: () => {
                return MathJax.startup.defaultPageReady();
            }
        }
    };

    function renderText() {
        const rawText = document.getElementById('input-text').value;
        const previewDiv = document.getElementById('preview');

        // Zabezpieczenie: zamiana nowych linii na <br> dla HTML, zachowując strukturę
        // Ale musimy uważać, żeby nie popsuć LaTeXa. 
        // Prosta metoda: escapowanie HTML, potem przywracanie LaTeX
        
        let formattedText = rawText
            .replace(/&/g, "&amp;")
            .replace(/</g, "&lt;")
            .replace(/>/g, "&gt;")
            .replace(/\n/g, "<br>");

        previewDiv.innerHTML = formattedText;

        // Wywołanie renderowania MathJax na elemencie podglądu
        MathJax.typesetPromise([previewDiv]).then(() => {
            // Sukces
            document.getElementById('status-msg').innerText = "Przetworzono! Możesz teraz skopiować.";
            setTimeout(() => document.getElementById('status-msg').innerText = "", 3000);
        }).catch((err) => {
            console.error(err);
            document.getElementById('status-msg').style.color = "red";
            document.getElementById('status-msg').innerText = "Błąd przetwarzania.";
        });
    }

    async function copyToWord() {
        const previewDiv = document.getElementById('preview');
        const statusMsg = document.getElementById('status-msg');

        if (!previewDiv.innerHTML.trim()) {
            statusMsg.style.color = "red";
            statusMsg.innerText = "Najpierw przetwórz tekst!";
            return;
        }

        try {
            // KROK KLUCZOWY:
            // MathJax 3 renderuje do HTML/CSS (chtml) domyślnie. Word woli MathML.
            // Ale MathJax w wersji 3 wstawia niewidoczny tag <math> (MathML) wewnątrz swojego wyjścia
            // dla celów dostępności. Word jest sprytny i potrafi to wykryć, jeśli skopiujemy jako HTML.
            
            // Tworzymy Blob typu text/html. To sprawia, że Word widzi to jako stronę internetową
            // i interpretuje formatowanie oraz ukryty MathML jako równania.
            
            const htmlContent = previewDiv.innerHTML;
            
            // Musimy opakować to w pełną strukturę HTML dla schowka, żeby Word wiedział co robić
            const blob = new Blob([
                `<html>
                  <head><meta charset='utf-8'></head>
                  <body>${htmlContent}</body>
                 </html>`
            ], { type: 'text/html' });

            // Również wersja tekstowa dla notatnika
            const textBlob = new Blob([previewDiv.innerText], { type: 'text/plain' });

            const data = [new ClipboardItem({
                "text/html": blob,
                "text/plain": textBlob
            })];

            await navigator.clipboard.write(data);

            statusMsg.style.color = "#28a745";
            statusMsg.innerText = "Skopiowano! Wklej teraz do Worda (Ctrl+V).";
        } catch (err) {
            console.error("Błąd kopiowania: ", err);
            
            // Fallback dla starszych przeglądarek lub braku uprawnień
            fallbackCopy(previewDiv);
        }
    }

    function fallbackCopy(element) {
        const range = document.createRange();
        range.selectNodeContents(element);
        const selection = window.getSelection();
        selection.removeAllRanges();
        selection.addRange(range);
        
        try {
            document.execCommand('copy');
            document.getElementById('status-msg').innerText = "Skopiowano (metoda zapasowa)!";
        } catch (err) {
            document.getElementById('status-msg').style.color = "red";
            document.getElementById('status-msg').innerText = "Nie udało się skopiować automatycznie. Zaznacz i skopiuj ręcznie.";
        }
        selection.removeAllRanges();
    }
</script>

</body>
</html>
