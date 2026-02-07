# Архитектура веб-приложения "Игра 101" 2026

## 🎯 Цель проекта
Создание современного, удобного и красивого веб-приложения для ведения счета в карточной игре "101" с интеграцией в Telegram Mini App.

## 🏗️ Архитектурные принципы

### 1. Паттерны проектирования
- **MVC Pattern**: Разделение логики, данных и представления
- **Observer Pattern**: Реактивное обновление UI при изменении состояния
- **Module Pattern**: Инкапсуляция кода в модули
- **Repository Pattern**: Абстракция работы с LocalStorage
- **State Machine**: Управление состояниями приложения

### 2. Структура модулей

```
Game101App/
├── core/
│   ├── GameEngine.js        - Основная игровая логика
│   ├── StateManager.js      - Управление состоянием приложения
│   └── EventBus.js          - Шина событий для Observer Pattern
├── storage/
│   ├── StorageManager.js    - Абстракция над LocalStorage
│   ├── GameRepository.js    - CRUD операции для игр
│   └── migrations.js        - Версионирование и миграции данных
├── ui/
│   ├── Router.js            - Навигация между экранами
│   ├── components/          - UI компоненты
│   │   ├── PlayerInput.js
│   │   ├── ScoreBoard.js
│   │   ├── RoundInput.js
│   │   ├── Chart.js
│   │   ├── GameHistory.js
│   │   └── Modal.js
│   └── animations.js        - Анимации и transitions
├── telegram/
│   ├── TelegramAPI.js       - Интеграция с Telegram WebApp
│   └── HapticFeedback.js    - Тактильные уведомления
└── utils/
    ├── validators.js        - Валидация данных
    ├── formatters.js        - Форматирование данных
    └── constants.js         - Константы приложения
```

## 💾 Структура данных LocalStorage

### Схема версии 1.0

```typescript
interface StorageSchema {
  version: "1.0";
  
  // Текущая активная игра (может быть null)
  currentGame: Game | null;
  
  // История завершенных игр
  completedGames: CompletedGame[];
  
  // Настройки пользователя
  settings: {
    theme: "auto" | "light" | "dark";
    hapticEnabled: boolean;
    soundEnabled: boolean;
  };
  
  // Статистика
  stats: {
    totalGames: number;
    totalRounds: number;
    playerStats: Record<string, PlayerStats>;
  };
}

interface Game {
  id: string;                    // UUID игры
  createdAt: number;             // timestamp создания
  updatedAt: number;             // timestamp последнего обновления
  players: Player[];             // Массив игроков
  rounds: Round[];               // История раундов
  status: "in_progress" | "paused";
}

interface Player {
  id: string;                    // UUID игрока
  name: string;                  // Имя игрока
  color: string;                 // Цвет для графика (HSL)
  score: number;                 // Текущий счет
  scoreHistory: number[];        // История счета по раундам
}

interface Round {
  roundNumber: number;           // Номер раунда
  timestamp: number;             // Время записи раунда
  points: Record<string, number>; // playerId -> очки
}

interface CompletedGame {
  id: string;
  createdAt: number;
  completedAt: number;
  duration: number;              // Длительность в мс
  players: Player[];
  rounds: Round[];
  winner: string | null;         // playerId победителя
  loser: string | null;          // playerId проигравшего
  totalRounds: number;
}

interface PlayerStats {
  gamesPlayed: number;
  gamesWon: number;
  gamesLost: number;
  totalPoints: number;
  avgPointsPerRound: number;
  bestScore: number;
  worstScore: number;
}
```

## 🎮 Игровая логика

### Правила игры 101

1. **Цель игры**: Первым набрать ровно 101 очко
2. **Начальный счет**: 0 очков
3. **Специальные правила**:
   - При достижении **ровно 50** очков → сброс до 25 (отсечка)
   - При достижении **ровно 101** очка → победа
   - При **превышении 101** очка → проигрыш
   - Можно иметь отрицательный счет

4. **Ввод очков**: Только целые числа (могут быть отрицательными)

### State Machine

```
States:
┌─────────────┐
│   SETUP     │ - Ввод имен игроков
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  PLAYING    │ - Активная игра
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  FINISHED   │ - Игра завершена
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  HISTORY    │ - Просмотр истории
└─────────────┘
```

## 🎨 UI/UX Архитектура

### Навигация (Screen Flow)

```
┌─────────────────┐
│  Welcome Screen │
│  - Продолжить   │ (если есть незавершенная игра)
│  - Новая игра   │
│  - История      │
│  - Настройки    │
└────────┬────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐  ┌──────────┐
│ Setup │  │ Game     │ (продолжить)
│ Screen│  │ Screen   │
└───┬───┘  └────┬─────┘
    │           │
    └─────┬─────┘
          ▼
    ┌──────────┐
    │  Game    │
    │  Screen  │
    │          │
    │ - Счет   │
    │ - Ввод   │
    │ - График │
    │ - Меню   │
    └────┬─────┘
         │
         ▼
    ┌──────────┐
    │ Finish   │
    │ Screen   │
    │          │
    │ - Итоги  │
    │ - График │
    │ - Заново │
    └──────────┘

┌──────────────┐
│   History    │
│   Screen     │
│              │
│ - Список игр │
│ - Детали     │
│ - Статистика │
│ - Очистить   │
└──────────────┘
```

### Компоненты UI

1. **NavigationBar** - Верхняя панель навигации с кнопкой назад
2. **PlayerSetup** - Форма добавления игроков с валидацией
3. **ScoreBoard** - Таблица текущих счетов с подсветкой лидера
4. **RoundInput** - Форма ввода очков за раунд
5. **LineChart** - График динамики счета с легендой
6. **GameCard** - Карточка игры в истории
7. **StatsPanel** - Панель статистики игрока
8. **ConfirmModal** - Модальное окно подтверждения действий
9. **BottomSheet** - Нижняя панель с действиями

## 🎨 Дизайн система 2026

### Цветовая палитра (Dark Theme Primary)

```css
:root {
  /* Base colors */
  --color-bg-primary: #0D0D0D;
  --color-bg-secondary: #1A1A1A;
  --color-bg-tertiary: #262626;
  --color-bg-elevated: #333333;
  
  /* Text colors */
  --color-text-primary: #FFFFFF;
  --color-text-secondary: #A0A0A0;
  --color-text-tertiary: #707070;
  --color-text-disabled: #4A4A4A;
  
  /* Accent colors */
  --color-accent-primary: #3B82F6;    /* Blue */
  --color-accent-success: #10B981;    /* Green */
  --color-accent-warning: #F59E0B;    /* Orange */
  --color-accent-error: #EF4444;      /* Red */
  --color-accent-purple: #8B5CF6;     /* Purple */
  
  /* Chart colors */
  --color-chart-1: #3B82F6;
  --color-chart-2: #10B981;
  --color-chart-3: #F59E0B;
  --color-chart-4: #EF4444;
  --color-chart-5: #8B5CF6;
  --color-chart-6: #EC4899;
  --color-chart-7: #14B8A6;
  --color-chart-8: #F97316;
  
  /* Borders */
  --color-border-subtle: #262626;
  --color-border-default: #333333;
  --color-border-strong: #4A4A4A;
  
  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.3);
  --shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.4);
  --shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.5);
  --shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.6);
  
  /* Spacing (8px base) */
  --space-1: 0.25rem;  /* 4px */
  --space-2: 0.5rem;   /* 8px */
  --space-3: 0.75rem;  /* 12px */
  --space-4: 1rem;     /* 16px */
  --space-5: 1.25rem;  /* 20px */
  --space-6: 1.5rem;   /* 24px */
  --space-8: 2rem;     /* 32px */
  --space-10: 2.5rem;  /* 40px */
  --space-12: 3rem;    /* 48px */
  --space-16: 4rem;    /* 64px */
  
  /* Border radius */
  --radius-sm: 0.375rem;   /* 6px */
  --radius-md: 0.5rem;     /* 8px */
  --radius-lg: 0.75rem;    /* 12px */
  --radius-xl: 1rem;       /* 16px */
  --radius-2xl: 1.5rem;    /* 24px */
  --radius-full: 9999px;
  
  /* Typography */
  --font-family: -apple-system, BlinkMacSystemFont, 'SF Pro Display', 'Segoe UI', 'Roboto', sans-serif;
  --font-size-xs: 0.75rem;    /* 12px */
  --font-size-sm: 0.875rem;   /* 14px */
  --font-size-base: 1rem;     /* 16px */
  --font-size-lg: 1.125rem;   /* 18px */
  --font-size-xl: 1.25rem;    /* 20px */
  --font-size-2xl: 1.5rem;    /* 24px */
  --font-size-3xl: 1.875rem;  /* 30px */
  --font-size-4xl: 2.25rem;   /* 36px */
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;
  
  /* Transitions */
  --transition-fast: 150ms cubic-bezier(0.4, 0, 0.2, 1);
  --transition-base: 200ms cubic-bezier(0.4, 0, 0.2, 1);
  --transition-slow: 300ms cubic-bezier(0.4, 0, 0.2, 1);
  
  /* Z-index layers */
  --z-base: 0;
  --z-dropdown: 1000;
  --z-sticky: 1100;
  --z-fixed: 1200;
  --z-modal-backdrop: 1300;
  --z-modal: 1400;
  --z-popover: 1500;
  --z-tooltip: 1600;
}
```

### Типографика

- **Display**: SF Pro Display, 36px, bold - для заголовков экранов
- **Heading 1**: SF Pro Display, 24px, semibold - для секций
- **Heading 2**: SF Pro Display, 20px, semibold - для подзаголовков
- **Body**: SF Pro Text, 16px, regular - основной текст
- **Caption**: SF Pro Text, 14px, regular - вспомогательный текст
- **Small**: SF Pro Text, 12px, regular - метки

## 🚀 Современные практики 2026

1. **Progressive Web App (PWA)**
   - Service Worker для offline режима
   - App manifest для установки
   
2. **Performance**
   - Lazy loading компонентов
   - Virtual scrolling для длинных списков
   - RequestAnimationFrame для анимаций
   - Debounce/throttle для input
   
3. **Accessibility (a11y)**
   - Semantic HTML
   - ARIA атрибуты
   - Keyboard navigation
   - Focus management
   - Screen reader support
   
4. **Mobile-first**
   - Touch-friendly (44px минимум для тапов)
   - Safe area insets
   - Viewport units
   - Gesture support (swipe, long-press)

5. **Error Handling**
   - Try-catch блоки
   - Graceful degradation
   - User-friendly сообщения
   - Logging для отладки

## 📱 Telegram WebApp Integration

### Используемые методы API

```javascript
// Инициализация
Telegram.WebApp.ready();
Telegram.WebApp.expand();
Telegram.WebApp.disableVerticalSwipes();

// Темы и цвета
Telegram.WebApp.setHeaderColor(color);
Telegram.WebApp.setBackgroundColor(color);
Telegram.WebApp.themeParams;

// Haptic feedback
Telegram.WebApp.HapticFeedback.impactOccurred('light');
Telegram.WebApp.HapticFeedback.notificationOccurred('success');
Telegram.WebApp.HapticFeedback.selectionChanged();

// Уведомления
Telegram.WebApp.showAlert(message);
Telegram.WebApp.showConfirm(message, callback);
Telegram.WebApp.showPopup(params);

// Кнопки
Telegram.WebApp.MainButton.setText(text);
Telegram.WebApp.MainButton.show();
Telegram.WebApp.MainButton.onClick(callback);
Telegram.WebApp.BackButton.show();
Telegram.WebApp.BackButton.onClick(callback);

// Closing behavior
Telegram.WebApp.enableClosingConfirmation();
Telegram.WebApp.close();
```

## ✅ Критерии качества

1. **Code Quality**
   - ESLint конфигурация
   - Чистый код, понятные имена
   - Комментарии для сложной логики
   - Нет дублирования кода

2. **User Experience**
   - Быстрая загрузка (< 2 сек)
   - Плавные анимации (60 FPS)
   - Понятный интерфейс
   - Обратная связь на действия

3. **Reliability**
   - Обработка всех ошибок
   - Валидация всех входных данных
   - Сохранение состояния при сбоях
   - Миграции данных при обновлениях

4. **Maintainability**
   - Модульная архитектура
   - Документация кода
   - Легко расширяемый
   - Тестируемый код
