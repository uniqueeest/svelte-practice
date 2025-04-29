<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { fly, fade } from 'svelte/transition';
  import { quintOut, cubicOut } from 'svelte/easing';
  import { browser } from '$app/environment';

  interface Bar {
    id: number;
    x: number;
    y: number;
    height: number;
    width: number;
    speed: number;
    caught: boolean;
    color: string;
    fallen: boolean;
  }

  interface ScorePopup {
    id: number;
    x: number;
    y: number;
    value: number;
  }

  let level = 1;
  let score = 0;
  let gameRunning = false;
  let gamePaused = false;
  let gameOver = false;
  let levelCompleted = false;
  let gameArea: HTMLDivElement;
  let gameWidth = 0;
  let gameHeight = 0;
  let bars: Bar[] = [];
  let animationId: number | null = null;
  let barCount = 3; // 시작 봉 개수
  let nextBarId = 0;
  let missedBars = 0;
  let maxMissedBars = 5; // 놓칠 수 있는 최대 봉 개수
  let scorePopups: ScorePopup[] = [];
  let nextPopupId = 0;
  let lastLevelUp = 0;
  let showLevelUp = false;
  let requiredCatchesToClearLevel = 10; // 레벨 클리어에 필요한 캐치 수
  let catchesInCurrentLevel = 0;

  // 레벨에 따른 봉 개수 계산
  $: {
    barCount = Math.min(3 + level - 1, 12); // 레벨 1은 3개, 최대 12개
  }

  // 레벨에 따른 배경색 효과
  $: levelBackground = getLevelBackground(level);

  // 색상 배열
  const colors = [
    '#3498db', // 파랑
    '#2ecc71', // 초록
    '#e74c3c', // 빨강
    '#f39c12', // 주황
    '#9b59b6', // 보라
    '#1abc9c', // 청록
    '#e67e22', // 주황빨강
    '#34495e', // 네이비
  ];

  // 레벨별 배경색 효과
  function getLevelBackground(level: number): string {
    switch (level) {
      case 1:
        return 'linear-gradient(to bottom, #f0f0f0, #e0e0e0)';
      case 2:
        return 'linear-gradient(to bottom, #e8f4f8, #d8e4e8)';
      case 3:
        return 'linear-gradient(to bottom, #e8f8e8, #d8e8d8)';
      case 4:
        return 'linear-gradient(to bottom, #f8f8e8, #e8e8d8)';
      case 5:
        return 'linear-gradient(to bottom, #f8e8e8, #e8d8d8)';
      case 6:
        return 'linear-gradient(to bottom, #f8e8f8, #e8d8e8)';
      case 7:
        return 'linear-gradient(to bottom, #e8e8f8, #d8d8e8)';
      case 8:
        return 'linear-gradient(to bottom, #e8f8f8, #d8e8e8)';
      case 9:
        return 'linear-gradient(to bottom, #f0e8d0, #e0d8c0)';
      case 10:
        return 'linear-gradient(to bottom, #f0d0d0, #e0c0c0)';
      default:
        return 'linear-gradient(to bottom, #f0f0f0, #e0e0e0)';
    }
  }

  // 게임 시작
  function startGame() {
    if (gameRunning && !gamePaused && !gameOver && !levelCompleted) return;

    if (levelCompleted) {
      // 다음 레벨로 진행
      level++;
      levelCompleted = false;
      catchesInCurrentLevel = 0;
      bars = [];
      scorePopups = [];

      // 초기 봉 생성
      createInitialBars();

      // 봉 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startRandomBarFalling();
        }
      }, 1000);
    } else if (gameOver || !gameRunning) {
      // 새 게임 시작
      gameRunning = true;
      gamePaused = false;
      gameOver = false;
      levelCompleted = false;
      score = 0;
      level = 1;
      missedBars = 0;
      catchesInCurrentLevel = 0;
      bars = []; // 기존 봉 모두 제거
      scorePopups = [];
      nextBarId = 0;
      nextPopupId = 0;

      // 초기 봉 생성
      createInitialBars();

      // 봉 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startRandomBarFalling();
        }
      }, 1000);
    } else if (gamePaused) {
      // 일시정지 해제
      gamePaused = false;
    }

    animate();
  }

  // 초기 봉 생성 (모두 상단에 위치)
  function createInitialBars() {
    const barWidth = 40; // 봉의 두께
    const spacing = (gameWidth - barCount * barWidth) / (barCount + 1);

    for (let i = 0; i < barCount; i++) {
      const height = Math.random() * 60 + 80; // 봉의 길이 (80~140px)
      const x = spacing + i * (barWidth + spacing);
      const color = colors[Math.floor(Math.random() * colors.length)];

      bars = [
        ...bars,
        {
          id: nextBarId++,
          x,
          y: 20, // 상단 여백
          height,
          width: barWidth,
          speed: 0, // 처음에는 움직이지 않음
          caught: false,
          color,
          fallen: false,
        },
      ];
    }
  }

  // 게임 일시정지
  function pauseGame() {
    if (!gameRunning || gameOver) return;

    gamePaused = true;
    if (animationId) {
      cancelAnimationFrame(animationId);
      animationId = null;
    }
  }

  // 게임 종료 처리
  function endGame() {
    gameRunning = false;
    gameOver = true;
    if (animationId) {
      cancelAnimationFrame(animationId);
      animationId = null;
    }
  }

  // 레벨 클리어 처리
  function levelClear() {
    levelCompleted = true;
    if (animationId) {
      cancelAnimationFrame(animationId);
      animationId = null;
    }

    // 레벨 클리어 시 잡은 봉이 서서히 사라지는 효과
    setTimeout(() => {
      bars = [];
    }, 500);
  }

  // 레벨 상승 표시
  function showLevelUpMessage() {
    showLevelUp = true;
    setTimeout(() => {
      showLevelUp = false;
    }, 2000);
  }

  // 랜덤하게 봉이 떨어지기 시작
  function startRandomBarFalling() {
    // 아직 떨어지지 않은 봉들 중에서 선택
    const notFallenBars = bars.filter((bar) => !bar.fallen && !bar.caught);

    if (notFallenBars.length === 0) return;

    // 랜덤하게 하나 선택
    const randomIndex = Math.floor(Math.random() * notFallenBars.length);
    const randomBar = notFallenBars[randomIndex];

    // 봉의 속도 설정 (레벨에 따라 증가)
    const speed = Math.random() * (1 + level * 0.3) + (1 + level * 0.2);

    // 선택된 봉이 떨어지기 시작
    bars = bars.map((bar) => (bar.id === randomBar.id ? { ...bar, speed, fallen: true } : bar));

    // 다음 봉이 떨어지기까지의 시간 (레벨이 높을수록 더 빨리)
    const nextFallDelay = Math.max(800 - level * 50, 300);

    // 다음 봉 떨어뜨리기 예약
    setTimeout(() => {
      if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
        startRandomBarFalling();
      }
    }, nextFallDelay);
  }

  // 봉 클릭 처리
  function handleBarClick(bar: Bar) {
    if (!gameRunning || gamePaused || gameOver || levelCompleted) return;

    if (!bar.caught && bar.fallen) {
      // 봉 캐치 성공
      const pointsEarned = Math.max(1, Math.floor(10 - bar.height / 20));
      score += pointsEarned;
      catchesInCurrentLevel++;

      // 점수 팝업 생성
      createScorePopup(bar.x + bar.width / 2, bar.y + bar.height / 2, pointsEarned);

      // 봉을 잡은 상태로 표시
      bars = bars.map((b) => (b.id === bar.id ? { ...b, caught: true } : b));

      // 레벨 클리어 체크
      if (catchesInCurrentLevel >= requiredCatchesToClearLevel) {
        levelClear();
      }
    }
  }

  // 점수 팝업 생성
  function createScorePopup(x: number, y: number, value: number) {
    const popup = {
      id: nextPopupId++,
      x,
      y,
      value,
    };

    scorePopups = [...scorePopups, popup];

    // 2초 후 제거
    setTimeout(() => {
      scorePopups = scorePopups.filter((p) => p.id !== popup.id);
    }, 2000);
  }

  // 게임 루프
  function animate() {
    if (gamePaused || gameOver || levelCompleted) return;

    // 봉 이동
    bars = bars.map((bar) => {
      // 이미 잡혔거나 아직 떨어지지 않은 봉은 움직이지 않음
      if (bar.caught || !bar.fallen) return bar;

      // 봉 이동
      const newY = bar.y + bar.speed;

      // 화면 밖으로 나갔는지 체크
      if (newY > gameHeight) {
        missedBars++;

        // 게임 오버 체크
        if (missedBars >= maxMissedBars) {
          endGame();
        }

        // 잡히지 않고 화면 밖으로 나간 봉 제거
        return { ...bar, caught: true };
      }

      return { ...bar, y: newY };
    });

    // 모든 봉이 떨어졌고 화면에 남은 봉이 없는 경우 새 봉 생성
    const activeBars = bars.filter((bar) => !bar.caught);
    const fallenBars = bars.filter((bar) => bar.fallen);

    if (activeBars.length === 0 && fallenBars.length === bars.length) {
      // 화면에 있는 모든 봉 제거
      bars = [];

      // 새로운 봉 생성
      createInitialBars();

      // 일정 시간 후 새 봉이 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startRandomBarFalling();
        }
      }, 1000);
    }

    if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
      animationId = requestAnimationFrame(animate);
    }
  }

  // 키보드 이벤트 처리
  function handleKeyDown(e: KeyboardEvent) {
    if (!gameRunning && !gameOver && !levelCompleted) return;

    if (e.key === 'Escape' || e.key === 'p' || e.key === 'P') {
      if (gameOver || levelCompleted) return;

      if (gamePaused) {
        startGame(); // 일시정지 해제
      } else {
        pauseGame(); // 일시정지
      }
    }
  }

  onMount(() => {
    // 게임 영역 크기 설정
    const rect = gameArea.getBoundingClientRect();
    gameWidth = rect.width;
    gameHeight = rect.height;

    // 키보드 이벤트 리스너 등록
    if (browser) {
      window.addEventListener('keydown', handleKeyDown);
    }
  });

  onDestroy(() => {
    if (animationId) {
      cancelAnimationFrame(animationId);
    }

    // 키보드 이벤트 리스너 제거
    if (browser) {
      window.removeEventListener('keydown', handleKeyDown);
    }
  });
</script>

<style>
  .container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
    padding: 20px;
    max-width: 800px;
    margin: 0 auto;
  }

  .game-header {
    display: flex;
    justify-content: space-between;
    width: 100%;
    align-items: center;
    background-color: #fff;
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  .score,
  .level,
  .progress,
  .missed-bars {
    font-size: 18px;
    font-weight: bold;
    margin-bottom: 5px;
  }

  .progress {
    color: #3498db;
  }

  .missed-bars {
    color: #e74c3c;
  }

  button {
    padding: 10px 20px;
    background-color: #4caf50;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
    transition: background-color 0.2s;
  }

  button:hover {
    background-color: #45a049;
  }

  button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }

  .game-area {
    position: relative;
    width: 100%;
    height: 500px;
    border: 2px solid #333;
    border-radius: 8px;
    overflow: hidden;
    transition: background 0.5s ease;
  }

  .bar {
    position: absolute;
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    transition:
      background-color 0.3s,
      opacity 0.3s,
      transform 0.3s;
    cursor: pointer;
    will-change: transform, opacity;
  }

  .bar.caught {
    opacity: 0.5;
    transform: scale(0.95);
  }

  .bar.falling {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }

  .score-popup {
    position: absolute;
    color: #4caf50;
    font-weight: bold;
    font-size: 18px;
    transform: translate(-50%, -50%);
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }

  .level-up {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: #ff5722;
    font-size: 48px;
    font-weight: bold;
    text-shadow: 0 0 10px rgba(255, 255, 255, 0.8);
    z-index: 10;
  }

  .game-message {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background-color: rgba(255, 255, 255, 0.9);
    padding: 20px 30px;
    border-radius: 10px;
    text-align: center;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 20;
  }

  .level-clear h2 {
    color: #3498db;
  }

  .game-over h2 {
    color: #e74c3c;
  }

  .instructions {
    text-align: center;
    margin-top: 20px;
    background-color: #fff;
    padding: 15px;
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
    width: 100%;
  }

  h2 {
    margin-bottom: 10px;
    color: #333;
  }
</style>

<svelte:head>
  <title>떨어지는 봉 잡기 게임</title>
</svelte:head>

<div class="container">
  <div class="game-header">
    <div>
      <div class="score">점수: {score}</div>
      <div class="level">레벨: {level}</div>
    </div>
    <div>
      <div class="progress">진행도: {catchesInCurrentLevel}/{requiredCatchesToClearLevel}</div>
      <div class="missed-bars">놓친 봉: {missedBars}/{maxMissedBars}</div>
    </div>
    <div class="controls">
      {#if levelCompleted}
        <button on:click={startGame}> 다음 레벨 </button>
      {:else if !gameRunning || gameOver}
        <button on:click={startGame}>
          {gameOver ? '다시 시작' : '게임 시작'}
        </button>
      {:else if gamePaused}
        <button on:click={startGame}> 계속하기 </button>
      {:else}
        <button on:click={pauseGame}> 일시정지 </button>
      {/if}
    </div>
  </div>

  <div class="game-area" bind:this={gameArea} style="background: {levelBackground};">
    {#each bars as bar (bar.id)}
      <div
        class="bar"
        class:caught={bar.caught}
        class:falling={bar.fallen && !bar.caught}
        on:click={() => handleBarClick(bar)}
        style="left: {bar.x}px; top: {bar.y}px; width: {bar.width}px; height: {bar.height}px; background-color: {bar.color};"
      ></div>
    {/each}

    {#each scorePopups as popup (popup.id)}
      <div
        class="score-popup"
        style="left: {popup.x}px; top: {popup.y}px;"
        transition:fly={{ y: -50, duration: 2000, easing: quintOut }}
      >
        +{popup.value}
      </div>
    {/each}

    {#if showLevelUp}
      <div class="level-up" transition:fade={{ duration: 800, easing: cubicOut }}>
        <span>레벨 {level}!</span>
      </div>
    {/if}

    {#if levelCompleted}
      <div class="game-message level-clear">
        <h2>레벨 {level} 클리어!</h2>
        <p>점수: {score}</p>
        <button on:click={startGame}>다음 레벨</button>
      </div>
    {/if}

    {#if gamePaused}
      <div class="game-message">
        <h2>일시정지</h2>
        <p>계속하려면 '계속하기' 버튼을 클릭하세요</p>
        <p>또는 'P' 키를 누르세요</p>
      </div>
    {/if}

    {#if gameOver}
      <div class="game-message game-over">
        <h2>게임 오버!</h2>
        <p>최종 점수: {score}</p>
        <p>최종 레벨: {level}</p>
        <button on:click={startGame}>다시 시작</button>
      </div>
    {/if}
  </div>

  <div class="instructions">
    <h2>게임 방법</h2>
    <p>떨어지는 봉을 클릭하여 잡으세요!</p>
    <p>더 짧은 봉을 잡을수록 더 많은 점수를 얻습니다.</p>
    <p>레벨 당 {requiredCatchesToClearLevel}개의 봉을 잡으면 다음 레벨로 넘어갑니다.</p>
    <p>레벨이 올라갈수록 봉이 더 빨리, 더 많이 떨어집니다.</p>
    <p>'P' 키나 'ESC' 키를 눌러 게임을 일시정지할 수 있습니다.</p>
    <p>봉을 {maxMissedBars}개 이상 놓치면 게임이 종료됩니다.</p>
  </div>
</div>
