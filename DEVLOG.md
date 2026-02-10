# XMage Accessibility Development Log

This document chronicles the development work done to add screen reader accessibility support to XMage, along with documentation improvements.

**Author**: Claude (AI Assistant)
**Branch**: `claude/add-claude-documentation-pekli`
**Date Range**: February 5-6, 2026
**Total Commits**: 8
**Total Lines Added**: ~1,800+

---

## Summary of Changes

| Commit | Description | Files Changed | Lines Added |
|--------|-------------|---------------|-------------|
| `c9f19e24` | CLAUDE.md project documentation | 1 | +407 |
| `7515d0f8` | Phase 1: Player panels & buttons | 5 | +247 |
| `4dd50754` | Phase 2: Card & zone accessibility | 5 | +149 |
| `a38f753f` | Phase 3: Keyboard navigation | 4 | +198 |
| `31fddc82` | Phase 4: Live announcements | 2 | +274 |
| `a1357131` | Phase 5: Deck editor & dialogs | 6 | +185 |
| `9995873e` | Accessibility hotkeys | 2 | +369 |
| `ca50cccf` | Windows 11 build guide | 1 | +356 |

---

## Commit 1: CLAUDE.md Project Documentation

**Commit**: `c9f19e24`
**Date**: 2026-02-05 10:20:46 UTC

### Purpose
Created comprehensive project documentation for AI assistants and new developers to understand the XMage codebase quickly.

### Changes
- **New file**: `CLAUDE.md` (407 lines)

### Content
- Project overview (28,000+ cards, 73,000+ reprints, dozens of formats)
- Repository structure (all modules explained)
- Build commands (Maven, Makefile)
- Test patterns (JUnit 5, CardTestPlayerBase framework)
- Card implementation guide with code examples
- Ability/effect class hierarchy
- Set definition structure
- Coding conventions and naming rules
- CI/CD pipeline documentation
- Plugin architecture overview
- Common development tasks

---

## Commit 2: Phase 1 — Player Panels & Buttons

**Commit**: `7515d0f8`
**Date**: 2026-02-05 12:40:40 UTC

### Purpose
Make the game UI navigable by screen reader users. Expose player stats and make action buttons keyboard-accessible.

### Problem Solved
- Player stats (life, hand count, library, etc.) were not exposed to assistive technology
- All skip/action buttons had `setFocusable(false)`, making them unreachable via keyboard
- Game prompts were not readable by screen readers

### Files Modified

| File | Changes |
|------|---------|
| `PlayerPanelExt.java` | Added `setAccessibleName()` to all stat labels (life, hand, library, graveyard, exile, poison, energy, experience, rad, command zone, mana pool). Added composite panel summary. |
| `GamePanel.java` | Removed `setFocusable(false)` from 10 buttons. Added accessible names to skip buttons, concede, macro toggle, switch hands, stop watching. |
| `HelperPanel.java` | Added accessible names to OK/Cancel/Yes/No/Done buttons. Exposed game prompt text via accessible name. |
| `FeedbackPanel.java` | Added accessible names to all 4 feedback buttons. Kept names synced with dynamic text changes. |
| `ACCESSIBILITY.md` | New file documenting Phase 1 completion, platform setup (NVDA/VoiceOver), and roadmap. |

### Technical Details
```java
// Example: Making a button accessible
btnSkipToNextTurn.setFocusable(true);  // Was false
btnSkipToNextTurn.getAccessibleContext()
    .setAccessibleName("Skip to next turn, F4");
```

---

## Commit 3: Phase 2 — Card & Zone Accessibility

**Commit**: `4dd50754`
**Date**: 2026-02-05 12:48:49 UTC

### Purpose
Make individual cards readable by screen readers with meaningful descriptions.

### Problem Solved
- Cards were visual-only components with no text representation for screen readers
- Zone containers (hand, battlefield, stack) had no accessible labels
- Card state (tapped, counters, summoning sickness) was not communicated

### Files Modified

| File | Changes |
|------|---------|
| `CardPanel.java` | Added `updateAccessibleName()` method building text from CardView: name, type, cost, P/T, state, rules. Handles face-down cards and stack abilities. Sets both name and description. |
| `Cards.java` | Added accessible name with zone name and card count in `setZone()` and `loadCards()`. |
| `BattlefieldPanel.java` | Added accessible name with permanent count at end of `update()`. |
| `CardArea.java` | Added accessible name with card count in `loadCards()`. |
| `ACCESSIBILITY.md` | Updated with Phase 2 documentation. |

### Technical Details
```java
// Example accessible name for a creature
"Llanowar Elves, Creature — Elf Druid, {G}, 1/1, tapped, 2 +1/+1 counters, {T}: Add {G}."

// Example for face-down card (no info leak)
"Face-down card"

// Example for stack ability
"Ability, When this creature enters, draw a card."
```

---

## Commit 4: Phase 3 — Keyboard Navigation

**Commit**: `a38f753f`
**Date**: 2026-02-05 16:21:52 UTC

### Purpose
Make the game fully playable via keyboard without requiring a mouse.

### Problem Solved
- Cards could not receive keyboard focus
- No visual focus indicator for sighted keyboard users
- No way to "click" a card via keyboard
- No navigation between cards in a zone

### Files Modified

| File | Changes |
|------|---------|
| `CardPanel.java` | Enabled `setFocusable(true)`. Added FocusListener for repaint. Added KeyListener for Enter/Space (click), arrow keys (navigate). Added `fireKeyboardClick()` for synthetic MouseEvent. Added `focusAdjacentCard()` for navigation. Added blue focus border in `paint()`. |
| `Cards.java` | Made cardArea a focus cycle root (`setFocusCycleRoot(true)`). |
| `BattlefieldPanel.java` | Made jPanel a focus cycle root. |
| `ACCESSIBILITY.md` | Added keyboard navigation quick reference table. |

### Keyboard Bindings Added

| Key | Action |
|-----|--------|
| Tab | Next card in zone |
| Shift+Tab | Previous card in zone |
| Ctrl+Tab | Jump to next zone/UI area |
| Left/Up | Previous card |
| Right/Down | Next card |
| Enter/Space | Click card (cast, activate, select) |

### Technical Details
```java
// Synthetic mouse click from keyboard
private void fireKeyboardClick() {
    MouseEvent synthetic = new MouseEvent(this, MouseEvent.MOUSE_CLICKED,
        System.currentTimeMillis(), 0, getWidth()/2, getHeight()/2, 1, false);
    callback.mouseClicked(synthetic, transferData);
}

// Focus indicator (bright blue border)
if (isFocusOwner()) {
    g2d.setColor(new Color(0, 180, 255));
    g2d.setStroke(new BasicStroke(3));
    g2d.drawRect(1, 1, getWidth()-3, getHeight()-3);
}
```

---

## Commit 5: Phase 4 — Live Announcements

**Commit**: `31fddc82`
**Date**: 2026-02-05 16:41:08 UTC

### Purpose
Proactively announce game state changes so screen reader users don't have to manually poll for updates.

### Problem Solved
- Phase/step changes were silent
- Priority passing was not announced
- Life total changes required manual checking
- Stack events (spells/abilities) were not communicated
- Combat assignments were visual-only

### Files Modified

| File | Changes |
|------|---------|
| `GamePanel.java` | Added hidden `accessibilityAnnouncer` JLabel. Added `announceToScreenReader()` method. Added `checkAndAnnounceGameStateChanges()` with state tracking. Added 8 tracking fields for change detection. First update suppressed to avoid flooding. |
| `ACCESSIBILITY.md` | Documented Phase 4, change detection table, how announcements work. |

### What Gets Announced

| Event | Example |
|-------|---------|
| Phase change | "Turn 3, Precombat Main" |
| Priority change | "Your priority" |
| Life change | "Your life: 20 -> 17" |
| Stack event | "Lightning Bolt on the stack" |
| Combat | "Grizzly Bears attacking, blocked by Soldier Token" |

### Technical Details
```java
// Hidden label for screen reader announcements
accessibilityAnnouncer = new JLabel();
accessibilityAnnouncer.setVisible(false);
pnlShortcuts.add(accessibilityAnnouncer);

// Fire property change to trigger announcement
private void announceToScreenReader(String message) {
    AccessibleContext ctx = accessibilityAnnouncer.getAccessibleContext();
    String oldName = ctx.getAccessibleName();
    ctx.setAccessibleName(message);
    ctx.firePropertyChange(AccessibleContext.ACCESSIBLE_NAME_PROPERTY,
                          oldName, message);
}
```

---

## Commit 6: Phase 5 — Deck Editor & Dialogs

**Commit**: `a1357131`
**Date**: 2026-02-06 00:50:00 UTC

### Purpose
Extend accessibility to the deck editor and modal dialogs.

### Problem Solved
- CardSelector filter buttons had `setFocusable(false)` (29 buttons!)
- Deck grid had no accessible description
- Modal dialogs (card selection, pile picking) were not labeled
- Search fields and lists lacked accessible names

### Files Modified

| File | Changes |
|------|---------|
| `CardSelector.java` | Added `initAccessibility()` setting focusable and accessible names on 29 filter buttons (color, type, rarity, view mode, search options), search field, table, combo boxes. |
| `DragCardGrid.java` | Added accessible name/description to main panel. Added dynamic description update in `updateCounts()` with card totals. |
| `ShowCardsDialog.java` | Added accessible name and description in `loadCards()`. |
| `PickChoiceDialog.java` | Added accessible name from choice message, labels for list and search. |
| `PickPileDialog.java` | Added accessible names to pile buttons and areas with card counts. |
| `ACCESSIBILITY.md` | Documented Phase 5 completion. |

### Buttons Made Accessible

| Category | Buttons |
|----------|---------|
| Colors | Red, Green, Blue, White, Black, Colorless |
| Types | Land, Creature, Artifact, Enchantment, Instant, Sorcery, Planeswalker |
| Rarity | Common, Uncommon, Rare, Mythic, Special |
| View | List view, Card grid view, Image mode |
| Search | Name, Type, Rules, Unique |

---

## Commit 7: Accessibility Announcement Hotkeys

**Commit**: `9995873e`
**Date**: 2026-02-06 01:12:40 UTC

### Purpose
Allow users to request on-demand announcements of game state via keyboard shortcuts.

### Problem Solved
- Users couldn't easily get a summary of current game state
- No way to hear all life totals at once
- Stack contents required navigating to each item
- Hand summary required focusing each card
- Battlefield required exploring each permanent

### Files Modified

| File | Changes |
|------|---------|
| `GamePanel.java` | Added `initAccessibilityHotkeys()` with Ctrl+Shift+key bindings. Added 7 announce/build methods. |
| `ACCESSIBILITY.md` | Documented hotkeys with examples. |

### Hotkeys Added

| Hotkey | Announcement |
|--------|--------------|
| Ctrl+Shift+G | Game state (turn, phase, step, active player, priority) |
| Ctrl+Shift+L | Life totals for all players |
| Ctrl+Shift+S | Stack contents (item count and names) |
| Ctrl+Shift+H | Hand summary (card count and names) |
| Ctrl+Shift+B | Battlefield summary (permanent counts by type per player) |
| Ctrl+Shift+C | Focused card full description |

### Technical Details
```java
// Register hotkey
KeyStroke ksGameState = KeyStroke.getKeyStroke(KeyEvent.VK_G,
    InputEvent.CTRL_MASK | InputEvent.SHIFT_MASK);
this.getInputMap(c).put(ksGameState, "ANNOUNCE_GAME_STATE");
this.getActionMap().put("ANNOUNCE_GAME_STATE", new AbstractAction() {
    @Override
    public void actionPerformed(ActionEvent e) {
        announceGameState();
    }
});

// Build comprehensive card description
private String buildCardDescription(CardView card) {
    StringBuilder sb = new StringBuilder();
    sb.append(card.getName());
    // ... type, cost, P/T, state, counters, rules
    return sb.toString();
}
```

---

## Commit 8: Windows 11 Build Guide

**Commit**: `ca50cccf`
**Date**: 2026-02-06 22:08:31 UTC

### Purpose
Provide comprehensive build instructions for Windows developers.

### Files Added

| File | Lines |
|------|-------|
| `BUILDING-WINDOWS.md` | 356 |

### Content
- Prerequisites (Java JDK, Maven, Git)
- Multiple installation methods (manual, Chocolatey, Scoop, winget)
- Environment variable configuration
- Step-by-step clone and build instructions
- Build options (skip tests, parallel, memory)
- Troubleshooting common issues
- IDE setup (IntelliJ, Eclipse, VS Code)
- Quick reference commands

---

## Files Created/Modified Summary

### New Files (3)
| File | Lines | Purpose |
|------|-------|---------|
| `CLAUDE.md` | 407 | AI/developer project documentation |
| `ACCESSIBILITY.md` | 369 | Screen reader accessibility documentation |
| `BUILDING-WINDOWS.md` | 356 | Windows 11 build guide |

### Modified Files (10)
| File | Purpose |
|------|---------|
| `GamePanel.java` | Buttons, announcements, hotkeys (+530 lines) |
| `CardPanel.java` | Card accessibility, focus, keyboard (+219 lines) |
| `CardSelector.java` | Filter button accessibility (+89 lines) |
| `PlayerPanelExt.java` | Player stats accessibility (+19 lines) |
| `HelperPanel.java` | Prompt accessibility (+14 lines) |
| `FeedbackPanel.java` | Button accessibility (+19 lines) |
| `Cards.java` | Zone accessibility (+12 lines) |
| `BattlefieldPanel.java` | Zone accessibility (+9 lines) |
| `CardArea.java` | Zone accessibility (+4 lines) |
| `DragCardGrid.java` | Deck grid accessibility (+13 lines) |
| `ShowCardsDialog.java` | Dialog accessibility (+5 lines) |
| `PickChoiceDialog.java` | Dialog accessibility (+7 lines) |
| `PickPileDialog.java` | Dialog accessibility (+10 lines) |

---

## Testing Recommendations

### Manual Testing Checklist

1. **NVDA on Windows**
   - Enable Java Access Bridge: `jabswitch /enable`
   - Restart system
   - Launch XMage, start game against AI
   - Verify player stats are announced
   - Tab through buttons, verify names
   - Navigate cards with arrows, verify announcements
   - Test all Ctrl+Shift hotkeys

2. **VoiceOver on macOS**
   - Enable VoiceOver (Cmd+F5)
   - Launch XMage
   - Use VO+Arrow keys to explore
   - Verify all UI elements are labeled

3. **Keyboard-Only Gameplay**
   - Play entire game without mouse
   - Cast spells with Enter
   - Select targets with arrow keys
   - Use F-keys for skip actions

---

## Architecture Decisions

### Why a Hidden JLabel for Announcements?
Java Swing's accessibility API requires a component to fire property change events. A hidden JLabel provides a lightweight anchor that doesn't affect layout but can trigger screen reader announcements via `ACCESSIBLE_NAME_PROPERTY` changes.

### Why Focus Cycle Roots for Zones?
Making each zone (hand, battlefield, stack) a focus cycle root means Tab cycles within the zone while Ctrl+Tab jumps between zones. This matches user expectations from web accessibility (Tab within widget, Ctrl+Tab between widgets).

### Why Ctrl+Shift for Hotkeys?
- Avoids conflicts with existing F-key shortcuts
- Avoids conflicts with OS shortcuts (Ctrl+C, etc.)
- Avoids conflicts with screen reader shortcuts
- Consistent modifier combination that's easy to remember

---

## Future Work

Items not yet implemented but would improve accessibility:

- [ ] Full keyboard navigation in DragCardGrid (deck editor)
- [ ] JTable column header accessible names
- [ ] Focus indicators in deck editor matching game focus indicators
- [ ] Keyboard-based drag-and-drop alternative
- [ ] Deck legality label accessibility
- [ ] Automated accessibility testing with javax.accessibility introspection

---

## References

- [Java Access Bridge](https://docs.oracle.com/javase/accessbridge/)
- [javax.accessibility API](https://docs.oracle.com/javase/8/docs/api/javax/accessibility/package-summary.html)
- [NVDA Screen Reader](https://www.nvaccess.org/)
- [Apple Accessibility](https://developer.apple.com/accessibility/)
- [WAI-ARIA Patterns](https://www.w3.org/WAI/ARIA/apg/patterns/)
