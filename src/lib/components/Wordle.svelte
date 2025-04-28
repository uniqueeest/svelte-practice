<script lang="ts">
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';

  // 게임 상태 관리
  const MAX_ATTEMPTS = 6;
  const WORD_LENGTH = 5;

  // 단어 목록 (영어와 한글 모두 추가)
  const WORDS = [
    // 영어 단어
    'APPLE',
    'BEACH',
    'CRANE',
    'DRINK',
    'EARTH',
    'FOCUS',
    'GRAPE',
    'HOUSE',
    'IVORY',
    'JUMBO',
    'KNIFE',
    'LEMON',
    'MUSIC',
    'NOISE',
    'OCEAN',
    'PLANT',
    'QUEEN',
    'RIVER',
    'SOLAR',
    'TOWER',
    'ULTRA',
    'VIVID',
    'WATCH',
    'XENON',
    'YACHT',
    'ZEBRA',
  ];

  type Status = 'correct' | 'present' | 'absent' | '';
  type KeyboardStatusType = Record<string, Status>;

  // 게임 상태
  const targetWord = writable('');
  const currentAttempt = writable(0);
  const guesses = writable(Array(MAX_ATTEMPTS).fill(''));
  const result = writable(Array(MAX_ATTEMPTS).fill(Array(WORD_LENGTH).fill('')));
  const gameOver = writable(false);
  const gameWon = writable(false);
  const currentGuess = writable('');
  const keyboardStatus = writable<KeyboardStatusType>({});

  // 게임 초기화
  function initGame() {
    // 랜덤 단어 선택
    const randomWord = WORDS[Math.floor(Math.random() * WORDS.length)];
    targetWord.set(randomWord);
    currentAttempt.set(0);
    guesses.set(Array(MAX_ATTEMPTS).fill(''));

    // 2D 배열을 각각 새로 생성하여 초기화
    const newResult = Array(MAX_ATTEMPTS)
      .fill(null)
      .map(() => Array(WORD_LENGTH).fill(''));
    result.set(newResult);

    gameOver.set(false);
    gameWon.set(false);
    currentGuess.set('');
    keyboardStatus.set({});
  }

  // 키보드 입력 처리
  function handleKeydown(event: KeyboardEvent) {
    if ($gameOver) return;

    if (event.key === 'Enter') {
      submitGuess();
    } else if (event.key === 'Backspace') {
      $currentGuess = $currentGuess.slice(0, -1);
    } else {
      const key = event.key.toUpperCase();
      if (/^[A-Z]$/.test(key) && $currentGuess.length < WORD_LENGTH) {
        $currentGuess += key;
      }
    }
  }

  // 추측 제출
  function submitGuess() {
    if ($currentGuess.length !== WORD_LENGTH) return;

    // 현재 추측 저장
    const newGuesses = [...$guesses];
    newGuesses[$currentAttempt] = $currentGuess;
    guesses.set(newGuesses);

    // 결과 계산 - 깊은 복사로 2D 배열 복사
    const newResult = $result.map((row) => [...row]);
    const guessResult = Array(WORD_LENGTH).fill('');
    const targetLetters = $targetWord.split('');
    const guessLetters = $currentGuess.split('');

    // 정확한 위치에 있는 글자 확인 (Green)
    for (let i = 0; i < WORD_LENGTH; i++) {
      if (guessLetters[i] === targetLetters[i]) {
        guessResult[i] = 'correct';
        targetLetters[i] = '';
      }
    }

    // 다른 위치에 있는 글자 확인 (Yellow)
    for (let i = 0; i < WORD_LENGTH; i++) {
      if (guessResult[i] === '') {
        const targetIndex = targetLetters.indexOf(guessLetters[i]);
        if (targetIndex !== -1) {
          guessResult[i] = 'present';
          targetLetters[targetIndex] = '';
        } else {
          guessResult[i] = 'absent';
        }
      }

      // 키보드 상태 업데이트
      const newKeyboardStatus = { ...$keyboardStatus };
      const key = guessLetters[i];
      const currentStatus = newKeyboardStatus[key] || '';
      const newStatus = guessResult[i] as Status;

      if (
        !currentStatus ||
        (currentStatus === 'absent' && (newStatus === 'present' || newStatus === 'correct')) ||
        (currentStatus === 'present' && newStatus === 'correct')
      ) {
        newKeyboardStatus[key] = newStatus;
      }

      keyboardStatus.set(newKeyboardStatus);
    }

    newResult[$currentAttempt] = guessResult;
    result.set(newResult);

    // 게임 승리 체크
    if ($currentGuess === $targetWord) {
      gameWon.set(true);
      gameOver.set(true);
    } else if ($currentAttempt === MAX_ATTEMPTS - 1) {
      // 게임 종료 체크
      gameOver.set(true);
    } else {
      // 다음 시도로 이동
      currentAttempt.update((n) => n + 1);
    }

    // 추측 초기화
    currentGuess.set('');
  }

  // 화면 클릭 키보드 처리
  function handleKeyClick(key: string) {
    if ($gameOver) return;

    if (key === 'ENTER') {
      submitGuess();
    } else if (key === '←') {
      $currentGuess = $currentGuess.slice(0, -1);
    } else if ($currentGuess.length < WORD_LENGTH) {
      $currentGuess += key;
    }
  }

  // 게임 재시작
  function restartGame() {
    initGame();
  }

  // 셀 클래스 계산 함수
  function getCellClass(status: string) {
    if (status === 'correct') return 'bg-green-500 text-white border-green-500';
    if (status === 'present') return 'bg-yellow-500 text-white border-yellow-500';
    if (status === 'absent') return 'bg-gray-500 text-white border-gray-500';
    return 'border-gray-300';
  }

  // 키보드 키 클래스 계산 함수
  function getKeyClass(status: string) {
    if (status === 'correct') return 'bg-green-500 text-white';
    if (status === 'present') return 'bg-yellow-500 text-white';
    if (status === 'absent') return 'bg-gray-500 text-white';
    return 'bg-gray-200';
  }

  // 안전하게 결과 배열에 접근하는 함수
  function getResultStatus(row: number, col: number): string {
    if (!$result || !$result[row] || !$result[row][col]) return '';
    return $result[row][col];
  }

  onMount(() => {
    initGame();
    window.addEventListener('keydown', handleKeydown);
    return () => {
      window.removeEventListener('keydown', handleKeydown);
    };
  });
</script>

<div class="flex flex-col items-center gap-8">
  <div class="flex flex-col gap-2">
    {#each Array(MAX_ATTEMPTS) as _, row}
      <div class="flex gap-2">
        {#each Array(WORD_LENGTH) as _, col}
          <div
            class="flex h-14 w-14 items-center justify-center border-2 text-2xl font-bold uppercase {getCellClass(
              getResultStatus(row, col),
            )}"
          >
            {#if row === $currentAttempt && col < $currentGuess.length}
              {$currentGuess[col]}
            {:else if $guesses[row] && col < $guesses[row].length}
              {$guesses[row][col]}
            {/if}
          </div>
        {/each}
      </div>
    {/each}
  </div>

  {#if $gameOver}
    <div class="mt-4 rounded-lg bg-gray-100 p-4 text-center">
      {#if $gameWon}
        <p>축하합니다! {$currentAttempt + 1}번 만에 맞췄습니다.</p>
      {:else}
        <p>아쉽네요. 정답은 {$targetWord} 였습니다.</p>
      {/if}
      <button class="mt-2 rounded bg-blue-500 px-4 py-2 font-bold text-white hover:bg-blue-600" on:click={restartGame}
        >다시 시작</button
      >
    </div>
  {/if}

  <div class="flex w-full max-w-lg flex-col gap-2">
    <div class="flex justify-center gap-1">
      {#each ['Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'I', 'O', 'P'] as key}
        <button
          class="h-12 min-w-10 rounded {getKeyClass(
            $keyboardStatus[key] || '',
          )} flex items-center justify-center font-bold"
          on:click={() => handleKeyClick(key)}
        >
          {key}
        </button>
      {/each}
    </div>

    <div class="flex justify-center gap-1">
      {#each ['A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L'] as key}
        <button
          class="h-12 min-w-10 rounded {getKeyClass(
            $keyboardStatus[key] || '',
          )} flex items-center justify-center font-bold"
          on:click={() => handleKeyClick(key)}
        >
          {key}
        </button>
      {/each}
    </div>

    <div class="flex justify-center gap-1">
      <button
        class="flex h-12 min-w-16 items-center justify-center rounded bg-gray-200 font-bold"
        on:click={() => handleKeyClick('ENTER')}>ENTER</button
      >
      {#each ['Z', 'X', 'C', 'V', 'B', 'N', 'M'] as key}
        <button
          class="h-12 min-w-10 rounded {getKeyClass(
            $keyboardStatus[key] || '',
          )} flex items-center justify-center font-bold"
          on:click={() => handleKeyClick(key)}
        >
          {key}
        </button>
      {/each}
      <button
        class="flex h-12 min-w-16 items-center justify-center rounded bg-gray-200 font-bold"
        on:click={() => handleKeyClick('←')}>←</button
      >
    </div>
  </div>
</div>
