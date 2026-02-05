# XMage Accessibility — Screen Reader Support

This document tracks the ongoing effort to make XMage accessible to screen reader
users, specifically **NVDA on Windows** and **VoiceOver on macOS**.

## Current Status: Phase 1 — Foundation (In Progress)

Phase 1 makes the core game state readable and game action buttons discoverable
by screen readers. It does not yet make cards selectable or the battlefield
navigable via keyboard alone.

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
- Custom-painted components (card images, mana symbols) will not be readable
  until Phase 2 adds `AccessibleContext` overrides to card panels.

## Files Modified in Phase 1

| File | Changes |
|------|---------|
| `Mage.Client/.../game/PlayerPanelExt.java` | Added `setAccessibleName()` in `setTextForLabel()` for all stat labels and their related icon components. Added composite accessible name to overall panel in `update()`. |
| `Mage.Client/.../game/GamePanel.java` | Removed `setFocusable(false)` from 10 buttons (skip, concede, macro, switch hands, stop watching). Added `setAccessibleName()` to each. |
| `Mage.Client/.../game/HelperPanel.java` | Added `setAccessibleName()` to all 4 buttons at init and on every text change. Exposed game prompt text via accessible name on panel and text area. |
| `Mage.Client/.../game/FeedbackPanel.java` | Added `setAccessibleName()` to panel and all 4 buttons at init. Kept names in sync in `setButtonState()` and `updateOptions()`. |

## Roadmap

### Phase 2 — Card Accessibility

Make individual cards readable by screen readers.

- Override `getAccessibleContext()` on `CardPanel` (in `org/mage/card/arcane/`)
  to return card name, type line, mana cost, power/toughness, abilities text,
  and current state (tapped, counters, attachments).
- Set `AccessibleRole` on card components so screen readers announce them as
  interactive items.
- Add accessible names to zone containers (Hand, Battlefield, Stack, Graveyard,
  Exile) including card counts.

**Key files**: `CardPanel.java`, `BattlefieldPanel.java`, `Cards.java`,
`CardArea.java`

### Phase 3 — Keyboard Navigation

Make the game playable without a mouse.

- Implement `FocusTraversalPolicy` per zone for arrow-key navigation between
  cards.
- Add Enter/Space to cast focused card or activate its ability.
- Add zone cycling (Tab to move between hand, battlefield, stack, opponent board).
- Keyboard-driven targeting: arrow keys to cycle legal targets, Enter to confirm.
- Keyboard-driven combat: select attackers/blockers from focused cards.
- Focus indicators (visual border/highlight on currently focused card).

**Key files**: `GamePanel.java`, `PlayAreaPanel.java`, `CardPanel.java`,
`BattlefieldPanel.java`

### Phase 4 — Live Announcements

Proactively announce game state changes so screen reader users can follow game
flow without polling.

- Announce priority changes: "Your priority — Main Phase 1"
- Announce phase transitions: "Combat — Declare Attackers"
- Announce stack events: "Opponent casts Lightning Bolt targeting you"
- Announce life total changes: "Your life: 20 → 17"
- Announce combat assignments: "Grizzly Bears attacking, blocked by Soldier Token"

Implementation: use `AccessibleContext.firePropertyChange()` on a live-region
component, or append to the game log `JTextPane` with proper accessible text
change events.

**Key files**: `GamePanel.java`, `FeedbackPanel.java`, `ChatPanelBasic.java`,
`GameTextPane.java`

### Phase 5 — Deck Editor & Dialogs

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
