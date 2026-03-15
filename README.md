<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <meta name="description" content="Super Tiger - Game platformer seru mengumpulkan dolar!">
    <title>Super Tiger</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body, html {
            margin: 0;
            padding: 0;
            width: 100%;
            height: 100%;
            background-color: #1e1b4b; /* Background gelap */
            touch-action: none; /* Mencegah zoom/scroll pada layar sentuh */
            overflow: hidden;
            font-family: sans-serif;
            -webkit-user-select: none;
            user-select: none;
        }
        #loading-screen {
            position: absolute;
            inset: 0;
            background-color: #1e1b4b;
            display: flex;
            justify-content: center;
            align-items: center;
            color: #fbbf24;
            font-size: 24px;
            font-weight: bold;
            z-index: 9999;
        }
        #game-container {
            position: relative;
            width: 100%;
            height: 100%;
            display: flex;
            justify-content: center;
            align-items: center;
            background-color: #3b82f6; /* Warna langit */
        }
        canvas {
            display: block;
            width: 100%;
            height: 100%;
            object-fit: cover;
            image-rendering: pixelated;
        }
        #ui-layer {
            position: absolute;
            inset: 0;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: space-between;
            padding: 1rem;
            z-index: 10;
        }
        .pointer-auto { pointer-events: auto; }
        
        /* Joystick Kiri */
        #analog-base {
            width: 120px;
            height: 120px;
            background: rgba(255, 255, 255, 0.2);
            border: 2px solid rgba(255, 255, 255, 0.5);
            border-radius: 50%;
            position: relative;
            backdrop-filter: blur(4px);
            touch-action: none;
        }
        #analog-stick {
            width: 50px;
            height: 50px;
            background: rgba(255, 255, 255, 0.9);
            border-radius: 50%;
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            box-shadow: 0 4px 6px rgba(0,0,0,0.3);
            pointer-events: none;
            transition: transform 0.05s linear;
        }

        /* Tombol Lompat Kanan */
        #btn-jump {
            width: 80px;
            height: 80px;
            background: rgba(34, 197, 94, 0.6);
            border: 3px solid rgba(134, 239, 172, 0.8);
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 36px;
            color: white;
            user-select: none;
            touch-action: none;
            box-shadow: 0 4px 10px rgba(0,0,0,0.3);
        }
        #btn-jump:active {
            background: rgba(34, 197, 94, 0.9);
            transform: scale(0.95);
        }
    </style>
</head>
<body>
    <!-- Layar Loading Awal -->
    <div id="loading-screen">MEMUAT GAME...</div>

    <div id="game-container">
        <!-- Canvas Game -->
        <canvas id="gameCanvas"></canvas>

        <!-- Lapisan Antarmuka (UI) -->
        <div id="ui-layer">
            <div class="flex justify-between items-start pointer-auto">
                <div class="bg-black/60 text-white font-mono text-xl md:text-2xl px-4 py-2 rounded-xl border-2 border-white/20 shadow-lg flex items-center gap-4">
                    <div>Lvl: <span id="level-display" class="text-yellow-400 font-bold">1</span></div>
                    <div>Dolar: <span id="score" class="text-green-400 font-bold">0</span> $</div>
                </div>
            </div>

            <div class="flex justify-between items-end w-full pb-4 px-2 pointer-auto mb-4 sm:hidden">
                <div id="analog-base"><div id="analog-stick"></div></div>
                <div id="btn-jump">△</div>
            </div>
        </div>

        <!-- Layar Overlay -->
        <div id="overlay" class="hidden absolute inset-0 bg-black/80 flex flex-col items-center justify-center text-white z-50 pointer-events-auto backdrop-blur-sm px-4 text-center">
            <h1 id="overlay-title" class="text-5xl md:text-6xl font-extrabold mb-4 text-yellow-400 drop-shadow-lg">GAME OVER</h1>
            <p id="overlay-desc" class="text-lg md:text-xl mb-8">Anda mengumpulkan <span id="final-score" class="text-green-400 font-bold">0</span> Dolar.</p>
            <button id="btn-restart" class="px-8 py-4 bg-blue-600 hover:bg-blue-500 text-white text-xl md:text-2xl font-bold rounded-2xl border-4 border-blue-400 shadow-[0_0_15px_rgba(59,130,246,0.5)] transition-transform active:scale-95">
                MAIN LAGI
            </button>
        </div>
    </div>

<script>
// Kode langsung dieksekusi agar tidak terhambat loading asset eksternal
(function() {
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    const LOGICAL_WIDTH = 800;
    const LOGICAL_HEIGHT = 560;
    const TILE_SIZE = 40;
    
    canvas.width = LOGICAL_WIDTH;
    canvas.height = LOGICAL_HEIGHT;

    // --- Aset Game ---
    const imgKarakter = new Image();
    imgKarakter.src = './1000074052.png'; 
    let gambarDimuat = false;
    imgKarakter.onload = () => { gambarDimuat = true; };

    // --- State & Variabel ---
    let frameId;
    let skor = 0;
    let kameraX = 0;
    let gameAktif = true;
    let currentLevel = 1;
    const MAX_LEVEL = 10;
    let overlayState = 'dead';
    const input = { left: false, right: false, up: false, down: false };

    let pemain = { 
        x: 50, y: 0, 
        width: 36, height: 36, 
        normalHeight: 36, duckHeight: 18, 
        isDucking: false, 
        vx: 0, vy: 0, 
        speed: 5.5, jumpPower: -10.5,
        grounded: false, arah: 1,
        jumpsLeft: 2, maxJumps: 2 
    };
    let platforms = [], dolars = [], musuhList = [], partikelList = [], pohonList = [];

    const daftarLevel = [
        [ // Level 1
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                       $$$                                      ",
            "                                      #####                                     ",
            "                                                                                ",
            "                  $$$                                         $ $ $             ",
            "                 #####                                        #####             ",
            "                                  E                                             ",
            "       E                      #########               E                       W ",
            "     #####                                         #######                  W W ",
            "                                                                          W W W ",
            "P                                                                       W W W W ",
            "################################################################################"
        ],
        [ // Level 2
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                 $$$                                            ",
            "                                #####                                 $ $       ",
            "               $$                                                     ###       ",
            "              ####                                                              ",
            "                                      E                                       W ",
            "                           ################        E                        W W ",
            "                                                 ######                   W W W ",
            "P                                                                       W W W W ",
            "################       ################################       ##################"
        ],
        [ // Level 3
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                            $$$                                 ",
            "                                           #####                                ",
            "                                                                                ",
            "               $$                                                     $ $ $     ",
            "              #####                         E                         #####   W ",
            "                             ^^        ###########       ^^                 W W ",
            "                            ####                        ####              W W W ",
            "P                                                                       W W W W ",
            "################################################################################"
        ],
        [ // Level 4
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                 $$                                             ",
            "                                ####                  $$$                       ",
            "                                                     #####                      ",
            "              $$                                                                ",
            "             ####                                                     $ $ $   W ",
            "                              E          ^^^                          ##### W W ",
            "                        #############   ######     E       ^^             W W W ",
            "                                                 #######  ####          W W W W ",
            "P                                                                     W W W W W ",
            "##################      ########################################      ##########"
        ],
        [ // Level 5
            "                                                                                ",
            "                                                                                ",
            "                                         $$$                                    ",
            "                                        #####                                   ",
            "                       $$                                                       ",
            "                      ####                                  $$$                 ",
            "                                                           #####                ",
            "                                                                                ",
            "             ####                                                             W ",
            "                                                    ####                    W W ",
            "                                 ####                                     W W W ",
            "                                                                        W W W W ",
            "P      ^^^             ^^^                   ^^^              ^^^     W W W W W ",
            "################################################################################"
        ],
        [ // Level 6
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "             ###########################   ############################         ",
            "                                $               $                               ",
            "                                                                                ",
            "               E    E      E           E                  E     E         $   W ",
            "             ###################       #############      #############   ### W ",
            "                                ^^                                          W W ",
            "                               ####                                       W W W ",
            "                                                                        W W W W ",
            "P      ^^^                                         ^^^                W W W W W ",
            "################################################################################"
        ],
        [ // Level 7
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                            $                  $                                ",
            "                           ###                ###                               ",
            "                                                                                ",
            "                 $                  $                  $                      W ",
            "                ###                ###                ###                   W W ",
            "                                                                          W W W ",
            "         ^^^            ^^^                ^^^                ^^^       W W W W ",
            "P       ######         ######             ######             ######   W W W W W ",
            "#####   ######   ###   ######     ###     ######    ###      ######   ##########"
        ],
        [ // Level 8
            "                                                                                ",
            "                                                                                ",
            "                                                                                ",
            "                                $$$                                             ",
            "                               #####                                            ",
            "                                                                                ",
            "                  $$                                          $$                ",
            "                 ####                                        ####               ",
            "                                                                                ",
            "                                               E                              W ",
            "                                            #######                         W W ",
            "                                  E                                       W W W ",
            "P                               #######                                 W W W W ",
            "######     ^^^^^^^^^^     ######       ######     ^^^^^^^^^^     ###############"
        ],
        [ // Level 9
            "                                                                                ",
            "                                                                                ",
            "                                $$$                                             ",
            "                               #####                                            ",
            "            E      $$                         E     $$                          ",
            "          ######  ####                      #####  ####                         ",
            "                                                                                ",
            "                                                                              W ",
            "                                                                            W W ",
            "                                                                          W W W ",
            "                           ###                            ###           W W W W ",
            "                                                                      W W W W W ",
            "P    ^^                 ^^             ^^              ^^           W W W W W W ",
            "########      ###      ########       #####     ###   #######      #############"
        ],
        [ // Level 10
            "                                                                                ",
            "                                            $$$$$                               ",
            "                                           #######                              ",
            "                                                                                ",
            "                                                                                ",
            "                       E                                                        ",
            "                     #####                                                      ",
            "                                     E                 E                        ",
            "             ###                  #######           #######                   W ",
            "                                                                            W W ",
            "                                                                          W W W ",
            "                                                                        W W W W ",
            "P   ^^^     ^^^       ^^^      ^^^        ^^^           
