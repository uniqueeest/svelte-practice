<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { fade } from 'svelte/transition';
  import { cubicOut } from 'svelte/easing';
  import { browser } from '$app/environment';
  import { cn } from '$lib/utils';
  import Button from '$lib/components/Button.svelte';

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

  const BAR_WIDTH = 40;
  const BAR_HEIGHT = 120;
  let level = 1;
  let gameRunning = false;
  let gamePaused = false;
  let gameOver = false;
  let levelCompleted = false;
  let gameArea: HTMLDivElement;
  let gameWidth = 0;
  let gameHeight = 0;
  let bars: Bar[] = [];
  let animationId: number | null = null;
  let nextBarId = 0;
  let missedBars = 0;
  let showLevelUp = false;
  let catchesInCurrentLevel = 0;
  let fallTimers: number[] = []; // 봉 떨어지는 타이머 저장

  // 레벨당 필요한 캐치 수와 봉 개수는 레벨에 따라 계산
  $: barsPerLevel = Math.min(2 + level, 12);
  $: requiredCatchesToClearLevel = barsPerLevel;
  // 동시에 떨어질 수 있는 최대 봉 개수 (레벨에 따라 증가)
  $: maxSimultaneousBars = Math.min(1 + Math.floor(level / 2), 5);
  // 봉이 떨어질 확률 (0~1 사이 값, 레벨이 높을수록 확률 증가)
  $: fallProbability = Math.min(0.3 + level * 0.05, 0.7);

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

  // 레벨별 배경색 효과 (순수 함수)
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

      // 타이머 초기화
      clearAllFallTimers();

      // 봉 초기화 - 기존 봉 완전히 제거
      bars = [];

      // 새 레벨에 맞는 봉 개수로 새로 생성
      const newBarsCount = Math.min(2 + level, 12);
      bars = createBars(gameWidth, newBarsCount);

      // 봉 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startMultipleBars(level);
        }
      }, 1000);
    } else if (gameOver || !gameRunning) {
      // 새 게임 시작
      gameRunning = true;
      gamePaused = false;
      gameOver = false;
      levelCompleted = false;
      level = 1;
      missedBars = 0;
      catchesInCurrentLevel = 0;
      bars = []; // 기존 봉 모두 제거
      nextBarId = 0;

      // 타이머 초기화
      clearAllFallTimers();

      // 레벨 1에 맞는 봉 개수로 생성
      const initialBarsCount = 3; // 레벨 1은 3개
      bars = createBars(gameWidth, initialBarsCount);

      // 봉 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startMultipleBars(level);
        }
      }, 1000);
    } else if (gamePaused) {
      // 일시정지 해제
      gamePaused = false;
    }

    animate();
  }

  // 게임 재시작
  function restartGame() {
    gameRunning = true;
    gamePaused = false;
    gameOver = false;
    levelCompleted = false;
    level = 1;
    missedBars = 0;
    catchesInCurrentLevel = 0;
    bars = []; // 기존 봉 모두 제거
    nextBarId = 0;

    // 타이머 초기화
    clearAllFallTimers();

    // 레벨 1에 맞는 봉 개수로 생성
    const initialBarsCount = 3; // 레벨 1은 3개
    bars = createBars(gameWidth, initialBarsCount);

    // 봉 떨어지기 시작
    setTimeout(() => {
      if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
        startMultipleBars(level);
      }
    }, 1000);

    animate();
  }

  // 봉 생성 함수 (순수 함수)
  function createBars(width: number, count: number): Bar[] {
    const spacing = (width - count * BAR_WIDTH) / (count + 1);
    const newBars: Bar[] = [];

    for (let i = 0; i < count; i++) {
      const x = spacing + i * (BAR_WIDTH + spacing);
      const color = colors[Math.floor(Math.random() * colors.length)];

      newBars.push({
        id: nextBarId++,
        x,
        y: 20,
        height: BAR_HEIGHT,
        width: BAR_WIDTH,
        speed: 0, // 처음에는 움직이지 않음
        caught: false,
        color,
        fallen: false,
      });
    }

    return newBars;
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

    // 타이머 초기화
    clearAllFallTimers();
  }

  // 레벨 클리어 처리
  function levelClear() {
    levelCompleted = true;
    if (animationId) {
      cancelAnimationFrame(animationId);
      animationId = null;
    }

    // 타이머 초기화
    clearAllFallTimers();

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

  // 모든 타이머 초기화
  function clearAllFallTimers() {
    for (const timer of fallTimers) {
      clearTimeout(timer);
    }
    fallTimers = [];
  }

  // 여러 개의 봉이 동시에 떨어지게 함
  function startMultipleBars(currentLevel: number) {
    // 현재 떨어지고 있는 봉의 개수 (fallen이 true이고 caught가 false인 봉)
    const currentlyFallingBars = bars.filter((bar) => bar.fallen && !bar.caught).length;

    // 모든 봉이 이미 떨어지고 있거나 잡힌 경우
    const notFallenBars = bars.filter((bar) => !bar.fallen && !bar.caught);
    if (notFallenBars.length === 0) {
      scheduleNextCheck(currentLevel);
      return;
    }

    // 랜덤 확률로 떨어질지 결정
    const shouldDropBar = Math.random() < fallProbability;

    if (shouldDropBar && currentlyFallingBars < maxSimultaneousBars) {
      // 랜덤하게 하나의 봉을 떨어뜨림
      startRandomBarFalling(currentLevel);
    }

    // 다음 체크 예약
    scheduleNextCheck(currentLevel);
  }

  // 다음 체크 시간 예약
  function scheduleNextCheck(currentLevel: number) {
    // 다음 봉이 떨어지기까지의 시간 (레벨이 높을수록 더 빨리)
    const nextFallDelay = Math.max(500 - currentLevel * 40, 200);

    const timer = setTimeout(() => {
      if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
        startMultipleBars(currentLevel);
      }
    }, nextFallDelay);

    fallTimers.push(timer);
  }

  // 랜덤하게 봉이 떨어지기 시작
  function startRandomBarFalling(currentLevel: number) {
    // 아직 떨어지지 않은 봉들 중에서 선택
    const notFallenBars = bars.filter((bar) => !bar.fallen && !bar.caught);

    if (notFallenBars.length === 0) return;

    // 랜덤하게 하나 선택
    const randomIndex = Math.floor(Math.random() * notFallenBars.length);
    const randomBar = notFallenBars[randomIndex];

    // 봉의 속도 설정 (레벨에 따라 증가)
    const speed = Math.random() * (1 + currentLevel * 0.3) + (1 + currentLevel * 0.2);

    // 선택된 봉이 떨어지기 시작
    bars = bars.map((bar) => (bar.id === randomBar.id ? { ...bar, speed, fallen: true } : bar));
  }

  // 봉 클릭 처리
  function handleBarClick(bar: Bar) {
    if (!gameRunning || gamePaused || gameOver || levelCompleted) return;

    if (!bar.caught && bar.fallen) {
      // 봉 캐치 성공
      catchesInCurrentLevel++;

      // 봉을 잡은 상태로 표시
      bars = bars.map((b) => (b.id === bar.id ? { ...b, caught: true } : b));

      // 레벨 클리어 체크
      if (catchesInCurrentLevel >= requiredCatchesToClearLevel) {
        levelClear();
      }
    }
  }

  // 봉 이동 처리 (순수 함수)
  function moveBar(bar: Bar, gameHeight: number): { bar: Bar; gameOver: boolean } {
    // 이미 잡혔거나 아직 떨어지지 않은 봉은 움직이지 않음
    if (bar.caught || !bar.fallen) {
      return { bar, gameOver: false };
    }

    // 봉 이동
    const newY = bar.y + bar.speed;

    // 화면 밖으로 나갔는지 체크
    if (newY > gameHeight) {
      // 잡히지 않고 화면 밖으로 나간 봉
      return {
        bar: { ...bar, caught: true },
        gameOver: true,
      };
    }

    return {
      bar: { ...bar, y: newY },
      gameOver: false,
    };
  }

  // 게임 루프
  function animate() {
    if (gamePaused || gameOver || levelCompleted) return;

    // 봉 이동
    let shouldEndGame = false;

    bars = bars.map((bar) => {
      const result = moveBar(bar, gameHeight);
      if (result.gameOver) {
        shouldEndGame = true;
      }
      return result.bar;
    });

    if (shouldEndGame) {
      endGame();
    }

    // 모든 봉이 떨어졌고 화면에 남은 봉이 없는 경우 새 봉 생성
    const activeBars = bars.filter((bar) => !bar.caught);
    const fallenBars = bars.filter((bar) => bar.fallen);

    if (activeBars.length === 0 && fallenBars.length === bars.length) {
      // 화면에 있는 모든 봉 제거
      bars = [];

      // 타이머 초기화
      clearAllFallTimers();

      // 현재 레벨에 맞는 정확한 수의 봉 생성
      const currentBarsCount = Math.min(2 + level, 12);
      bars = createBars(gameWidth, currentBarsCount);

      // 일정 시간 후 새 봉이 떨어지기 시작
      setTimeout(() => {
        if (gameRunning && !gamePaused && !gameOver && !levelCompleted) {
          startMultipleBars(level);
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

    // 타이머 초기화
    clearAllFallTimers();

    // 키보드 이벤트 리스너 제거
    if (browser) {
      window.removeEventListener('keydown', handleKeyDown);
    }
  });
</script>

<style>
  .bar.caught {
    opacity: 0.5;
    transform: scale(0.95);
  }

  .bar.falling {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
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

<div class={cn('flex flex-col items-center gap-5', 'mx-auto p-5 lg:max-w-[800px]')}>
  <div class={cn('flex w-full items-center justify-between', 'rounded-lg bg-white p-5 shadow-md')}>
    <h2 class={cn('text-lg font-bold')}>레벨: {level}</h2>
    <div class={cn('flex flex-col items-center gap-2')}>
      <div class={cn('text-sm font-bold', 'text-blue-500')}>
        진행도: {catchesInCurrentLevel}/{requiredCatchesToClearLevel}
      </div>
      <div class={cn('text-sm font-bold', 'text-red-500')}>하나라도 놓치면 게임오버!</div>
    </div>
    <div class={cn('flex gap-2')}>
      {#if levelCompleted}
        <Button onclick={startGame}>다음 레벨</Button>
      {:else if !gameRunning || gameOver}
        <Button onclick={startGame}>게임 시작</Button>
      {:else if gamePaused}
        <Button color="secondary" onclick={startGame}>계속하기</Button>
        <Button onclick={restartGame}>다시 시작</Button>
      {:else}
        <Button color="secondary" onclick={pauseGame}>일시정지</Button>
        <Button onclick={restartGame}>다시 시작</Button>
      {/if}
    </div>
  </div>

  <div
    class={cn(
      'relative',
      'h-[500px] w-full',
      'overflow-hidden rounded-lg border border-[#333]',
      'transition-background ease duration-500',
    )}
    bind:this={gameArea}
    style="background: {levelBackground};"
  >
    {#each bars as bar (bar.id)}
      <button
        class={cn(
          'absolute',
          'h-[120px] w-10',
          'rounded-lg',
          'box-shadow',
          'cursor-pointer',
          'will-change-opacity will-change-transform',
          bar.caught && 'opacity-50',
          bar.fallen && !bar.caught && 'box-shadow',
        )}
        aria-label="봉"
        onkeydown={() => {}}
        onclick={() => handleBarClick(bar)}
        style="left: {bar.x}px; top: {bar.y}px; background-color: {bar.color};"
      ></button>
    {/each}

    {#if showLevelUp}
      <div class="level-up" transition:fade={{ duration: 800, easing: cubicOut }}>
        <span>레벨 {level}!</span>
      </div>
    {/if}

    {#if levelCompleted}
      <div class="game-message level-clear">
        <h2>레벨 {level} 클리어!</h2>
        <Button color="primary" onclick={startGame}>다음 레벨</Button>
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
        <p>최종 레벨: {level}</p>
        <Button onclick={startGame}>다시 시작</Button>
      </div>
    {/if}
  </div>

  <div class="instructions">
    <h2>게임 방법</h2>
    <p>떨어지는 봉을 클릭하여 잡으세요!</p>
    <p>레벨 당 {requiredCatchesToClearLevel}개의 봉을 모두 잡으면 다음 레벨로 넘어갑니다.</p>
    <p>레벨이 올라갈수록 봉이 더 빨리, 더 많이 떨어집니다.</p>
    <p>'P' 키나 'ESC' 키를 눌러 게임을 일시정지할 수 있습니다.</p>
    <p>봉을 하나라도 놓치면 게임이 종료됩니다!</p>
  </div>
</div>
