# XMage Accessibility — Screen Reader Support

This document tracks the ongoing effort to make XMage accessible to screen reader
users, specifically **NVDA on Windows** and **VoiceOver on macOS**.

## Current Status: Phase 4 — Live Announcements (Complete)

Phases 1 (Foundation), 2 (Card Accessibility), 3 (Keyboard Navigation), and
4 (Live Announcements) are complete. Screen readers now proactively announce
game state changes including phase transitions, priority, life totals, stack
events, and combat.

### What Works After Phase 1

- **Player panel stats** (life, hand count, library, graveyard, exile, poison,
  energy, experience, rad, command zone, mana pool) are all exposed via
  `AccessibleContext` names. A screen reader will announce e.g. "life: 17" or
  "White mana: 3" when focus or virtual cursor reaches these elements.

- **Overall player panel summary**: each player panel has a composite accessible
  name like "PlayerA (you), life: 20, hand: 7, library: 53".

- **Skip/action buttons** (Cancel Skip, Skip to Next Turn, Skip to End of Turn,
  Skip to Next Main Phase, Skip to Your Turn, Skip to End Step Before Your Turn,
  Skip Stack, Concede, Toggle Macro, Switch Hands, Stop Watching) are now
  **focusable via Tab/Shift+Tab** and have accessible names. Previously all of
  these had `setFocusable(false)`.

- **Feedback/helper panel buttons** (OK, Cancel, Yes, No, Done, Close Game,
  Special, Undo) have accessible names that update dynamically as the game
  changes prompt modes.

- **Game prompt text** is exposed to screen readers via the HelperPanel's
  accessible name (HTML stripped). When the game asks "Do you want to cast
  Lightning Bolt?", screen readers can read that prompt.

### F-Key Shortcuts (unchanged, already existed)

These work globally during a game when chat input is not focused:

| Key | Action |
|-----|--------|
| F2 | Confirm / OK / Yes / Done |
| F3 | Cancel all skips |
| F4 | Skip to next turn |
| F5 | Skip to end of turn |
| F6 | End turn, skip stack |
| F7 | Skip to next main phase |
| F8 | Toggle macro recording |
| F9 | Skip to your turn |
| F10 | Skip until stack resolved |
| F11 | Skip to end step before your turn |
| F12 | Toggle chat input focus |
| Alt+E | Enlarge hovered card |
| Alt+1 | Use first mana ability (hold) |

## Platform Setup

### Windows (NVDA)

XMage uses Java Swing, which requires **Java Access Bridge** to communicate with
Windows screen readers.

1. Ensure you are running JDK 9+ (Access Bridge is bundled). JDK 8 requires
   a separate Access Bridge install from Oracle.
2. Enable Java Access Bridge:
   ```
   jabswitch /enable
   ```
   Or set the system property in `accessibility.properties`:
   ```
   assistive_technologies=com.sun.java.accessibility.AccessBridge
   ```
3. Restart your system after enabling.
4. Launch XMage. NVDA should begin reading Swing components.

**Recommended JDK**: 17 or 21 — the Access Bridge is most stable on these.

### macOS (VoiceOver)

Swing integrates with VoiceOver through Apple's native accessibility bridge.
No additional setup is needed beyond enabling VoiceOver (Cmd+F5). However:

- VoiceOver support for Swing has historically been inconsistent across macOS
  versions. Test on the latest macOS for best results.
- Card panels now have `AccessibleContext` names (added in Phase 2), but
  custom-painted decorations (foil effects, mana symbol images) remain visual-only.

## Files Modified in Phase 1

| File | Changes |
|------|---------|
| `Mage.Client/.../game/PlayerPanelExt.java` | Added `setAccessibleName()` in `setTextForLabel()` for all stat labels and their related icon components. Added composite accessible name to overall panel in `update()`. |
| `Mage.Client/.../game/GamePanel.java` | Removed `setFocusable(false)` from 10 buttons (skip, concede, macro, switch hands, stop watching). Added `setAccessibleName()` to each. |
| `Mage.Client/.../game/HelperPanel.java` | Added `setAccessibleName()` to all 4 buttons at init and on every text change. Exposed game prompt text via accessible name on panel and text area. |
| `Mage.Client/.../game/FeedbackPanel.java` | Added `setAccessibleName()` to panel and all 4 buttons at init. Kept names in sync in `setButtonState()` and `updateOptions()`. |

## What Works After Phase 2

- **Individual cards are readable**: each card component (`CardPanel`) exposes
  a concise accessible name built from the `CardView` data. For example:
  `"Lightning Bolt, Instant, {R}, Lightning Bolt deals 3 damage to any target."`

- **Permanent state is included**: for battlefield permanents, the accessible
  name includes tapped/untapped status, summoning sickness, and counters.
  Example: `"Llanowar Elves, Creature — Elf Druid, {G}, 1/1, tapped, 2 +1/+1 counters"`

- **Face-down cards**: announced as "Face-down card" without leaking hidden info.

- **Stack abilities**: announced as "Ability, [rules text]".

- **Choosable/selected state**: when a card is a valid target or already selected
  during targeting, this is appended to the accessible name.

- **Full card text as description**: the `AccessibleDescription` contains the
  complete tooltip text (name, cost, type, P/T, all rules, set info) for users
  who want to read the full Oracle text.

- **Zone containers have accessible names**: the Hand, Stack, Battlefield, and
  card selection dialogs announce their zone name and card count. Example:
  `"Hand zone, 7 cards"`, `"Battlefield, 4 permanents"`,
  `"Card selection, 3 cards"`.

## Files Modified in Phase 2

| File | Changes |
|------|---------|
| `Mage.Client/.../arcane/CardPanel.java` | Added `updateAccessibleName()` method that builds concise screen-reader text from `CardView` data (name, type, cost, P/T, state, rules). Called from constructor and `update(CardView)`. Sets both `accessibleName` and `accessibleDescription`. Added `CounterView` import. |
| `Mage.Client/.../cards/Cards.java` | Added accessible name to zone in `setZone()` and updated with card count in `loadCards()`. |
| `Mage.Client/.../game/BattlefieldPanel.java` | Added accessible name with permanent count at end of `update()`. |
| `Mage.Client/.../cards/CardArea.java` | Added accessible name with card count in `loadCards()`. |

## What Works After Phase 3

- **Cards are focusable**: every card component (`CardPanel`) is now focusable
  via `setFocusable(true)`. Screen readers announce the card's accessible name
  when it receives focus.

- **Visual focus indicator**: a bright blue (RGB 0, 180, 255) 3px border is drawn
  around the focused card so sighted keyboard users can see which card is active.

- **Enter/Space to click a card**: pressing Enter or Space on a focused card
  triggers the same action as a mouse click. This works for casting spells from
  hand, activating abilities, selecting targets, and confirming choices.

- **Arrow key navigation within a zone**: Left/Up moves focus to the previous card
  in the zone; Right/Down moves to the next card. Cards are sorted by visual
  position (top-to-bottom, then left-to-right).

- **Zone-scoped Tab traversal**: each zone container (hand, stack, battlefield)
  is a **focus cycle root**. This means:
  - **Tab / Shift+Tab** cycles between cards within the current zone.
  - **Ctrl+Tab / Ctrl+Shift+Tab** jumps to the next/previous zone or UI area
    (standard Swing focus cycle root behavior).

### Keyboard Navigation Quick Reference

| Key | Context | Action |
|-----|---------|--------|
| Tab | In a zone | Move to next card in the zone |
| Shift+Tab | In a zone | Move to previous card in the zone |
| Ctrl+Tab | Anywhere | Jump to the next zone / UI area |
| Ctrl+Shift+Tab | Anywhere | Jump to the previous zone / UI area |
| Left / Up | On a card | Focus the previous card in the zone |
| Right / Down | On a card | Focus the next card in the zone |
| Enter / Space | On a card | Click the card (cast, activate, select target) |
| F2 | Global | Confirm / OK / Done |
| F3–F11 | Global | Skip/concede shortcuts (see Phase 1 section) |

### How Keyboard Card Clicking Works

When the user presses Enter or Space on a focused card, `CardPanel.fireKeyboardClick()`
creates a synthetic `MouseEvent` and calls `callback.mouseClicked()` with the same
`TransferData` (component, card, gameId) that a real mouse click would provide.
This means all game actions that work via mouse click also work via keyboard:

1. **Casting from hand**: focus a card in the Hand zone, press Enter.
2. **Activating abilities**: focus a permanent on the Battlefield, press Enter.
3. **Selecting targets**: when the game prompts for a target, choosable cards are
   marked (announced as "choosable" by screen readers). Arrow to the target, Enter.
4. **Confirming choices**: the existing F2 shortcut also works for OK/Done prompts.

## Files Modified in Phase 3

| File | Changes |
|------|---------|
| `Mage.Client/.../arcane/CardPanel.java` | Enabled `setFocusable(true)` (was commented out). Added `FocusListener` to repaint on focus change. Added `KeyListener` for Enter/Space (click), Left/Right/Up/Down (navigate). Added `fireKeyboardClick()` method that creates synthetic `MouseEvent` and calls callback. Added `focusAdjacentCard()` method that finds sibling `MageCard` components by position and transfers focus. Added bright blue focus indicator border in `paint()`. |
| `Mage.Client/.../cards/Cards.java` | Made `cardArea` a focus cycle root (`setFocusCycleRoot(true)`) so Tab cycles within hand/stack zone cards. |
| `Mage.Client/.../game/BattlefieldPanel.java` | Made `jPanel` a focus cycle root (`setFocusCycleRoot(true)`) so Tab cycles within battlefield permanents. |

## What Works After Phase 4

- **Phase/step transitions announced**: when the game moves to a new phase or
  step, screen readers announce it. Examples:
  - `"Turn 3, Precombat Main"`
  - `"Declare Attackers"`
  - `"End Turn"`

- **Priority changes announced**: when priority passes between players, screen
  readers announce who has priority:
  - `"Your priority"` (when it's the local player's turn to act)
  - `"OpponentName's priority"`

- **Life total changes announced**: when any player's life total changes, the
  old and new values are announced:
  - `"Your life: 20 -> 17"`
  - `"OpponentName's life: 15 -> 12"`

- **Stack events announced**: when new spells or abilities are added to the
  stack, they are announced:
  - `"Lightning Bolt on the stack"`
  - `"Llanowar Elves's ability on the stack"` (for abilities, the rules text
    is used if available)

- **Combat announced**: when combat groups change (attackers declared, blockers
  assigned), a summary is announced:
  - `"Combat: Grizzly Bears attacking OpponentName"`
  - `"Combat: Serra Angel attacking OpponentName blocked by Soldier Token"`

### How Live Announcements Work

A hidden `JLabel` component (`accessibilityAnnouncer`) is added to the shortcuts
panel in `GamePanel`. Each time `updateGame()` runs, the method
`checkAndAnnounceGameStateChanges()` compares the current `GameView` with
previously saved state and builds a list of changes. These are joined into a
single string and set as the label's accessible name via
`AccessibleContext.firePropertyChange(ACCESSIBLE_NAME_PROPERTY, ...)`.

Screen readers monitoring the accessibility tree detect the property change and
speak the new text. This works on:
- **NVDA (Windows)**: via Java Access Bridge property change notifications
- **VoiceOver (macOS)**: via the native accessibility bridge

**First-update suppression**: on game start, the first call to
`checkAndAnnounceGameStateChanges()` only saves state without announcing,
to avoid flooding the screen reader with the initial game state.

### What Is Tracked for Change Detection

| State | Field | Compared Per Update |
|-------|-------|---------------------|
| Phase step | `previousStep` (`PhaseStep` enum) | Enum identity (`!=`) |
| Turn number | `previousTurn` (`int`) | Integer inequality |
| Priority player | `previousPriorityPlayer` (`String`) | String `.equals()` |
| Life totals | `previousLifeTotals` (`Map<UUID, Integer>`) | Per-player integer comparison |
| Stack objects | `previousStackIds` (`Set<UUID>`) | Set difference (new UUIDs) |
| Combat groups | `previousCombatGroupCount` (`int`) | Count change |

## Files Modified in Phase 4

| File | Changes |
|------|---------|
| `Mage.Client/.../game/GamePanel.java` | Added `javax.accessibility.AccessibleContext` import. Added hidden `accessibilityAnnouncer` JLabel in `initComponents()`. Added `announceToScreenReader(String)` method that fires `ACCESSIBLE_NAME_PROPERTY` change events. Added `checkAndAnnounceGameStateChanges(GameView)` that detects phase, priority, life, stack, and combat changes. Added `saveAccessibilityState(GameView)` to persist state between updates. Added 8 tracking fields. Called from end of `updateGame()`. |

## Roadmap

### Phase 5 — Deck Editor & Dialogs (Next)

- Ensure all modal dialogs (card selection, choice prompts, pick dialogs) have
  proper tab order and accessible labels.
- Add keyboard navigation to `DragCardGrid` (image-based deck view).
- The table-based card views (`CardsList`, `CardSelector`) already use `JTable`
  which has built-in accessibility — verify they expose column headers and cell
  values correctly.

**Key files**: `DeckEditorPanel.java`, `DragCardGrid.java`, `CardSelector.java`,
`ShowCardsDialog.java`, all `MageDialog` subclasses

## Testing

### Manual Testing

1. **NVDA (Windows)**: Enable Java Access Bridge, launch XMage, start a game
   against AI. Use Tab to navigate buttons, use NVDA browse mode (Insert+Space
   to toggle) to explore player panels. Verify stats are announced.

2. **VoiceOver (macOS)**: Enable VoiceOver (Cmd+F5), launch XMage, use VO+Arrow
   keys to explore the interface. Verify player panels and buttons are read.

3. **Keyboard navigation (Phase 3)**: Start a game against AI.
   - Press Ctrl+Tab until focus enters the Hand zone. Verify a bright blue border
     appears around the focused card.
   - Press Right/Left arrows to move between cards in hand. Verify the blue border
     moves and the screen reader announces each card's name, type, and cost.
   - Press Enter on a card in hand to cast it. Verify the game responds as if the
     card was clicked.
   - Press Ctrl+Tab to jump to the Battlefield. Use arrows to navigate permanents.
   - Press Enter on a permanent to activate abilities or select it as a target
     when prompted.

4. **Live announcements (Phase 4)**: Start a game against AI with a screen reader
   active.
   - Verify that phase transitions are announced (e.g., "Precombat Main",
     "Declare Attackers") as you pass through turn phases.
   - Verify priority changes are announced ("Your priority" when it's your turn
     to act).
   - Cast a spell and verify the stack event is announced (e.g., "Lightning Bolt
     on the stack").
   - Take damage and verify life total changes are announced (e.g., "Your life:
     20 -> 17").
   - Enter combat and verify attacker/blocker announcements.
   - Verify that the initial game load does NOT flood with announcements (first
     update is suppressed).

### Automated Testing (Future)

Consider adding `javax.accessibility` introspection tests to verify components
expose proper `AccessibleContext` data:

```java
// Example: verify player panel has accessible name
AccessibleContext ctx = playerPanel.getAccessibleContext();
assertThat(ctx.getAccessibleName()).contains("life:");
assertThat(ctx.getAccessibleName()).contains("hand:");
```

## References

- [Java Access Bridge documentation](https://docs.oracle.com/javase/accessbridge/)
- [Java Accessibility API (javax.accessibility)](https://docs.oracle.com/javase/8/docs/api/javax/accessibility/package-summary.html)
- [NVDA Java support](https://www.nvaccess.org/)
- [Apple Accessibility for Java](https://developer.apple.com/accessibility/)
