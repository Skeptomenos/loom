# Discovery: Should the TUI implement a dedicated `CountdownWidget` or extend the existing `Spinner` to support countdown mode?

> Category: Follow-up (Patterns)
> Discovered: 2026-01-16
> Confidence: High

## Answer

**The TUI should implement a dedicated `CountdownWidget` rather than extending the existing `Spinner`.** While extending the Spinner might seem simpler, the two widgets have fundamentally different purposes, state models, and rendering requirements. A dedicated widget provides cleaner separation of concerns and follows the existing crate-per-widget pattern in the codebase.

The investigation reveals:

1. **Different semantic purposes**: A Spinner indicates indeterminate progress ("something is happening"), while a Countdown indicates determinate progress ("X seconds remaining"). Conflating these creates a confusing API.

2. **Different state models**: `SpinnerState` tracks only a frame index for animation. A countdown needs `start_time: Instant`, `duration: Duration`, and methods like `remaining()` and `is_expired()`. These are incompatible state shapes.

3. **Different rendering logic**: Spinner renders an animated character + optional label. Countdown needs to render remaining time (formatted as "Xs" or "X:XX"), optionally with a progress bar, and handle the "expired" state.

4. **Existing crate-per-widget pattern**: The codebase already has separate crates for each widget (`loom-tui-widget-spinner`, `loom-tui-widget-status-bar`, `loom-tui-widget-tool-panel`, etc.). A new `loom-tui-widget-countdown` follows this established pattern.

5. **Composition over extension**: The TUI can compose both widgets when needed (e.g., show a spinner during LLM "Thinking..." and a countdown during retry waits) without either widget becoming bloated.

## Evidence

**Spinner has minimal state** (`crates/loom-tui-widget-spinner/src/lib.rs:46-55`):
```rust
#[derive(Clone, Debug, Default)]
pub struct SpinnerState {
    frame: usize,  // Only tracks animation frame
}

impl SpinnerState {
    pub fn tick(&mut self) {
        self.frame = self.frame.wrapping_add(1);
    }
}
```

**Spinner renders animation + label** (`crates/loom-tui-widget-spinner/src/lib.rs:103-124`):
```rust
impl StatefulWidget for Spinner {
    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State) {
        let frames = self.kind.frames();
        let frame_char = frames[state.frame % frames.len()];
        let text = match &self.label {
            Some(label) => format!("{} {}", frame_char, label),
            None => frame_char.to_string(),
        };
        buf.set_string(x, y, &text, self.text_style);
    }
}
```

**Crate-per-widget pattern is established**:
```
crates/loom-tui-widget-spinner/
crates/loom-tui-widget-status-bar/
crates/loom-tui-widget-tool-panel/
crates/loom-tui-widget-thread-list/
crates/loom-tui-widget-message-list/
crates/loom-tui-widget-input-box/
crates/loom-tui-widget-modal/
crates/loom-tui-widget-markdown/
crates/loom-tui-widget-header/
crates/loom-tui-widget-scrollable/
```

**ToolPanel shows duration pattern** (`crates/loom-tui-widget-tool-panel/src/lib.rs:43-49`):
```rust
pub struct ToolExecution {
    pub name: String,
    pub status: ToolStatus,
    pub output: Option<String>,
    pub duration_ms: Option<u64>,  // Duration display already exists
}
```

**Key Files:**
- `crates/loom-tui-widget-spinner/src/lib.rs` — Current Spinner implementation (simple, animation-focused)
- `crates/loom-tui-widget-tool-panel/src/lib.rs` — Example of duration display in widgets
- `crates/loom-tui-app/src/app.rs` — Shows how widgets are composed in the main app
- `crates/loom-tui-component/src/lib.rs` — Component trait pattern for reference

## Recommended Implementation

### New Crate: `loom-tui-widget-countdown`

```rust
// crates/loom-tui-widget-countdown/src/lib.rs

use std::time::{Duration, Instant};
use ratatui::prelude::*;
use ratatui::widgets::StatefulWidget;

#[derive(Clone, Copy, Debug, Default)]
pub enum CountdownStyle {
    #[default]
    Seconds,      // "15s"
    MinutesSeconds, // "1:30"
    Verbose,      // "1 minute 30 seconds remaining"
}

#[derive(Clone, Debug)]
pub struct CountdownState {
    start: Instant,
    duration: Duration,
}

impl CountdownState {
    pub fn new(duration: Duration) -> Self {
        Self {
            start: Instant::now(),
            duration,
        }
    }

    pub fn remaining(&self) -> Duration {
        self.duration.saturating_sub(self.start.elapsed())
    }

    pub fn is_expired(&self) -> bool {
        self.start.elapsed() >= self.duration
    }

    pub fn progress(&self) -> f64 {
        let elapsed = self.start.elapsed().as_secs_f64();
        let total = self.duration.as_secs_f64();
        if total == 0.0 { 1.0 } else { (elapsed / total).min(1.0) }
    }

    /// Reset the countdown with a new duration
    pub fn reset(&mut self, duration: Duration) {
        self.start = Instant::now();
        self.duration = duration;
    }
}

#[derive(Clone, Debug)]
pub struct Countdown {
    label: Option<String>,
    expired_label: Option<String>,
    style: CountdownStyle,
    text_style: Style,
    show_progress_bar: bool,
    progress_bar_width: u16,
}

impl Countdown {
    pub fn new() -> Self {
        Self {
            label: None,
            expired_label: None,
            style: CountdownStyle::default(),
            text_style: Style::default(),
            show_progress_bar: false,
            progress_bar_width: 20,
        }
    }

    pub fn label(mut self, label: impl Into<String>) -> Self {
        self.label = Some(label.into());
        self
    }

    pub fn expired_label(mut self, label: impl Into<String>) -> Self {
        self.expired_label = Some(label.into());
        self
    }

    pub fn style(mut self, style: CountdownStyle) -> Self {
        self.style = style;
        self
    }

    pub fn text_style(mut self, style: Style) -> Self {
        self.text_style = style;
        self
    }

    pub fn with_progress_bar(mut self, width: u16) -> Self {
        self.show_progress_bar = true;
        self.progress_bar_width = width;
        self
    }

    fn format_remaining(&self, remaining: Duration) -> String {
        let secs = remaining.as_secs();
        match self.style {
            CountdownStyle::Seconds => format!("{}s", secs),
            CountdownStyle::MinutesSeconds => {
                let mins = secs / 60;
                let secs = secs % 60;
                if mins > 0 {
                    format!("{}:{:02}", mins, secs)
                } else {
                    format!("{}s", secs)
                }
            }
            CountdownStyle::Verbose => {
                let mins = secs / 60;
                let secs = secs % 60;
                match (mins, secs) {
                    (0, s) => format!("{} seconds remaining", s),
                    (1, 0) => "1 minute remaining".to_string(),
                    (m, 0) => format!("{} minutes remaining", m),
                    (1, s) => format!("1 minute {} seconds remaining", s),
                    (m, s) => format!("{} minutes {} seconds remaining", m, s),
                }
            }
        }
    }
}

impl Default for Countdown {
    fn default() -> Self {
        Self::new()
    }
}

impl StatefulWidget for Countdown {
    type State = CountdownState;

    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State) {
        if area.width == 0 || area.height == 0 {
            return;
        }

        if state.is_expired() {
            let text = self.expired_label.as_deref().unwrap_or("Ready");
            buf.set_string(area.x, area.y, text, self.text_style);
            return;
        }

        let remaining = state.remaining();
        let time_str = self.format_remaining(remaining);

        let text = match &self.label {
            Some(label) => format!("{} {}", label, time_str),
            None => time_str,
        };

        let mut x = area.x;
        buf.set_string(x, area.y, &text, self.text_style);

        if self.show_progress_bar && area.height > 1 {
            x = area.x;
            let y = area.y + 1;
            let progress = state.progress();
            let filled = (self.progress_bar_width as f64 * progress) as u16;
            let empty = self.progress_bar_width - filled;

            buf.set_string(x, y, "[", self.text_style);
            x += 1;
            buf.set_string(x, y, &"=".repeat(filled as usize), self.text_style);
            x += filled;
            buf.set_string(x, y, &" ".repeat(empty as usize), self.text_style);
            x += empty;
            buf.set_string(x, y, "]", self.text_style);
        }
    }
}
```

### Usage in App

```rust
// In AppState
pub struct AppState {
    // ... existing fields ...
    pub countdown_state: Option<CountdownState>,
}

// In App::render
if let Some(countdown_state) = &mut self.state.countdown_state {
    let countdown = Countdown::new()
        .label("Rate limited. Retrying in")
        .expired_label("Retrying now...")
        .style(CountdownStyle::Seconds)
        .text_style(self.theme.text.warning);
    
    let countdown_area = Rect { /* ... */ };
    frame.render_stateful_widget(countdown, countdown_area, countdown_state);
}
```

## Implications

1. **Create new crate `loom-tui-widget-countdown`**: Follow the established pattern with its own `Cargo.toml`, `src/lib.rs`, and tests.

2. **Keep Spinner unchanged**: The Spinner widget remains focused on its single purpose (indeterminate progress animation).

3. **Add to storybook**: Create `crates/loom-tui-storybook/src/stories/countdown.rs` for visual testing.

4. **AppState needs countdown tracking**: Add `countdown_state: Option<CountdownState>` to track active retry countdowns.

5. **Countdown is time-based, not tick-based**: Unlike Spinner which needs `tick()` calls, Countdown uses `Instant::now()` internally and updates automatically on each render.

6. **Consider i18n**: The `Verbose` style strings should eventually use `loom-common-i18n` for translation.

## Follow-up Questions

(None - this discovery completes the countdown widget design question)

## Related Discoveries

- [[025-cli-retry-wait-ux]] — Established need for countdown timer, not just spinner
- [[026-cli-retry-wait-cancellation]] — Cancellation handling for retry waits
- [[016-shared-agent-runtime-analysis]] — Proposed AgentRuntime would trigger countdown display
