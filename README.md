# PN-lot-qty-scanner
Scan labels
<html lang="he">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>PN LOT QTY Scanner</title>

https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.jsscript>

<style>
body{
    font-family:Arial,sans-serif;
    margin:0;
    padding:20px;
    background:#f5f5f5;
}

h1{
    text-align:center;
    color:#1976d2;
}

.card{
    background:#fff;
    padding:15px;
    margin-bottom:15px;
    border-radius:10px;
    box-shadow:0 2px 5px rgba(0,0,0,.1);
}

#reader{
    width:100%;
}

table{
    width:100%;
    border-collapse:collapse;
}

th,td{
    border:1px solid #ccc;
    padding:8px;
    text-align:left;
}

th{
    background:#1976d2;
    color:white;
}

button{
    padding:10px 15px;
    border:none;
    border-radius:5px;
    background:#1976d2;
    color:white;
    cursor:pointer;
}

button:hover{
    background:#1259a7;
}

.success{
    color:green;
    font-weight:bold;
}

.warning{
    color:red;
    font-weight:bold;
}
</style>
</head>
<body>

<h1>PN / LOT / QTY Scanner</h1>

<div class="card">
    <div id="reader"></div>
</div>

<div class="card">
    <div>נ“¦ Boxes: <span id="boxCount">0</span></div>
    <div>נ“ Total Qty: <span id="totalQty">0</span></div>
    <div id="status"></div>
</div>

<div class="card">
    <h3>Last Scan</h3>

    <div>PN: <span id="pn">-</span></div>
    <div>LOT: <span id="lot">-</span></div>
    <div>QTY: <span id="qty">-</span></div>
</div>

<div class="card">
    <button onclick="exportCSV()">Export CSV</button>
    <button onclick="clearSession()">Clear Session</button>
</div>

<div class="card">

<table>
    <thead>
        <tr>
            <th>PN</th>
            <th>LOT</th>
            <th>QTY</th>
            <th>TIME</th>
        </tr>
    </thead>
    <tbody id="scanTable"></tbody>
</table>

</div>

<script>

let scans = [];
let scannedBarcodes = new Set();

function saveLocal() {
    localStorage.setItem(
        "warehouse_scans",
        JSON.stringify(scans)
    );
}

function loadLocal() {

    const data =
        localStorage.getItem("warehouse_scans");

    if (!data) return;

    scans = JSON.parse(data);

    scans.forEach(item => {

        scannedBarcodes.add(item.raw);

        addRow(item);

    });

    updateTotals();
}

function addRow(scan) {

    const tr =
        document.createElement("tr");

    tr.innerHTML = `
        <td>${scan.pn}</td>
        <td>${scan.lot}</td>
        <td>${scan.qty}</td>
        <td>${scan.time}</td>
    `;

    document
        .getElementById("scanTable")
        .appendChild(tr);
}

function updateTotals() {

    document.getElementById("boxCount")
        .innerText = scans.length;

    let totalQty = 0;

    scans.forEach(x => {
        totalQty += parseInt(x.qty) || 0;
    });

    document.getElementById("totalQty")
        .innerText = totalQty;
}

function processScan(text) {

    text = text.trim();

    if(scannedBarcodes.has(text)){

        document.getElementById("status").innerHTML =
            '<span class="warning">Already Scanned</span>';

        return;
    }

    const parts = text.split("|");

    if(parts.length < 3){

        document.getElementById("status").innerHTML =
            '<span class="warning">Invalid Format</span>';

        return;
    }

    const scan = {

        pn: parts[0].trim(),
        lot: parts[1].trim(),
        qty: parts[2].trim(),
        raw: text,
        time: new Date().toLocaleString()

    };

    scans.push(scan);

    scannedBarcodes.add(text);

    document.getElementById("pn").innerText =
        scan.pn;

    document.getElementById("lot").innerText =
        scan.lot;

    document.getElementById("qty").innerText =
        scan.qty;

    addRow(scan);

    updateTotals();

    saveLocal();

    document.getElementById("status").innerHTML =
        '<span class="success">ג“ Saved</span>';

    if(navigator.vibrate){
        navigator.vibrate(100);
    }
}

async function startScanner() {

    try {

        const cameras =
            await Html5Qrcode.getCameras();

        if (!cameras.length) {

            alert("No camera found");
            return;
        }

        const cameraId = cameras[0].id;

        const html5QrCode =
            new Html5Qrcode("reader");

        await html5QrCode.start(
            cameraId,
            {
                fps: 10,
                qrbox: {
                    width: 250,
                    height: 250
                }
            },
            processScan,
            () => {}
        );

    } catch(err) {

        console.error(err);

        alert(
            "Unable to access camera"
        );
    }
}

function exportCSV() {

    let csv =
        "PN,LOT,QTY,TIME\n";

    scans.forEach(item => {

        csv +=
        `"${item.pn}","${item.lot}","${item.qty}","${item.time}"\n`;

    });

    const blob =
        new Blob([csv],
        {type:"text/csv"});

    const link =
        document.createElement("a");

    link.href =
        URL.createObjectURL(blob);

    link.download =
        "Warehouse_Scans.csv";

    link.click();
}

function clearSession() {

    if(!confirm("Delete all scans?"))
        return;

    scans = [];
    scannedBarcodes.clear();

    localStorage.removeItem(
        "warehouse_scans"
    );

    document.getElementById("scanTable")
        .innerHTML = "";

    updateTotals();
}

loadLocal();
startScanner();

</script>

</body>
</html>
