# CLAUDE.md - XMage Development Guide

## Project Overview

XMage (Magic, Another Game Engine) is an open-source Java implementation of Magic: The Gathering with full rules enforcement. It supports 28,000+ unique cards, 73,000+ reprints, and dozens of game formats (Standard, Modern, Commander, Pioneer, Legacy, Pauper, Brawl, Oathbreaker, etc.). Players can compete against AI opponents or other players online.

**License**: MIT
**Version**: 1.4.58
**Java Target**: 1.8 (compatible with Java 8 through 21)

## Repository Structure

```
mage/
├── Mage/                       # Core game engine (rules, abilities, effects, game logic)
├── Mage.Client/                # Swing-based desktop GUI client
├── Mage.Common/                # Shared utilities, constants, database models
├── Mage.Server/                # Game server and networking
├── Mage.Server.Console/        # Command-line server interface
├── Mage.Sets/                  # Card implementations and set definitions
│   └── src/mage/
│       ├── cards/[a-z]/        # Individual card classes (alphabetized by first letter)
│       └── sets/               # Set definitions (ExpansionSet subclasses)
├── Mage.Server.Plugins/        # Plugin modules (27 total)
│   ├── Mage.Game.*             # Game format plugins (TwoPlayerDuel, Commander, Brawl, etc.)
│   ├── Mage.Player.*           # Player implementations (AI, Human, DraftBot)
│   ├── Mage.Deck.*             # Deck format validators (Constructed, Limited)
│   └── Mage.Tournament.*       # Tournament types (Swiss, Elimination, Draft, Sealed)
├── Mage.Tests/                 # Test suite (JUnit 5)
├── Mage.Verify/                # Card data verification tools
├── Mage.Reports/               # JaCoCo code coverage aggregation
├── Mage.Plugins/               # Counter plugin and extensibility
├── Utils/                      # Build scripts, card generation tools, data files
├── .github/                    # GitHub Actions workflows, Dependabot config
└── .travis/                    # Travis CI configuration
```

## Build Commands

The project uses **Apache Maven 3.x** as its build system.

```bash
# Full build (skip tests)
mvn install package -DskipTests

# Run tests only
mvn test -B

# Clean build artifacts
mvn clean

# Full clean build with packaging
make install              # Runs: clean, build, package

# Package client and server ZIPs
make package              # Outputs to deploy/ directory

# Build only (no packaging)
make build
```

**Build output locations:**
- Server: `Mage.Server/target/mage-server.zip`
- Client: `Mage.Client/target/mage-client.zip`

## Running Tests

```bash
# Run all tests
mvn test -B

# Run tests with suppressed data collection logs (CI mode)
mvn test -B -Dxmage.dataCollectors.printGameLogs=false

# Run a specific test class
mvn test -pl Mage.Tests -Dtest=org.mage.test.turnmod.ExtraTurnsTest

# Run a specific test method
mvn test -pl Mage.Tests -Dtest=org.mage.test.turnmod.ExtraTurnsTest#test_EmrakulMustGiveExtraTurn_OnOwnTurn
```

Tests are located in `Mage.Tests/src/test/java/org/mage/test/` and use JUnit 5 (Jupiter) with AssertJ assertions.

## Key Technologies

| Technology | Version | Purpose |
|-----------|---------|---------|
| Java | 8+ (target 1.8) | Primary language |
| Maven | 3.x | Build system |
| JUnit 5 | 5.8.1 | Testing framework |
| AssertJ | 3.21.0 | Test assertions |
| Swing | - | Desktop GUI |
| H2 Database | 1.4.197 | Embedded database |
| ORMLite | 5.7 | ORM |
| Google Guava | 33.4.8-jre | Collections/utilities |
| Gson | 2.13.2 | JSON processing |
| SLF4J + Reload4j | 2.0.17 | Logging |
| JaCoCo | 0.8.11 | Code coverage |
| Protocol Buffers | 3.x | Data serialization |

## Card Implementation Guide

### File Location

Cards are organized alphabetically by first letter of the class name:
`Mage.Sets/src/mage/cards/[a-z]/CardClassName.java`

Set definitions live in: `Mage.Sets/src/mage/sets/SetName.java`

### Card Class Structure

Every card class follows this pattern:

```java
package mage.cards.w;

import java.util.UUID;
import mage.MageInt;
import mage.abilities.keyword.FlyingAbility;
import mage.cards.CardImpl;
import mage.cards.CardSetInfo;
import mage.constants.CardType;
import mage.constants.SubType;

public final class WildGriffin extends CardImpl {

    public WildGriffin(UUID ownerId, CardSetInfo setInfo) {
        super(ownerId, setInfo, new CardType[]{CardType.CREATURE}, "{2}{W}");
        this.subtype.add(SubType.GRIFFIN);
        this.power = new MageInt(2);
        this.toughness = new MageInt(2);

        // Flying
        this.addAbility(FlyingAbility.getInstance());
    }

    private WildGriffin(final WildGriffin card) {
        super(card);
    }

    @Override
    public WildGriffin copy() {
        return new WildGriffin(this);
    }
}
```

**Required elements for every card:**
1. `public final class` declaration extending `CardImpl` (or a specialized base class)
2. Public constructor taking `(UUID ownerId, CardSetInfo setInfo)`
3. `super()` call with card types and mana cost string
4. Private copy constructor taking `(final ClassName card)`
5. `copy()` method returning new instance via copy constructor

### Card Type Variants

| Card Type | Base Class | Special Setup |
|-----------|-----------|---------------|
| Creature | `CardImpl` | Set `power`, `toughness` via `MageInt` |
| Planeswalker | `CardImpl` | Call `setStartingLoyalty(N)`, use `LoyaltyAbility` |
| Adventure | `AdventureCard` | Call `finalizeAdventure()` at end of constructor |
| Transform/DFC | `TransformingDoubleFacedCard` | Use `getLeftHalfCard()` / `getRightHalfCard()` |
| Battle | `CardImpl` | Call `setStartingDefense(N)` |

### Adding Abilities

```java
// Keyword abilities (use singleton getInstance())
this.addAbility(FlyingAbility.getInstance());
this.addAbility(TrampleAbility.getInstance());

// Triggered abilities
this.addAbility(new EntersBattlefieldTriggeredAbility(
    new DrawCardSourceControllerEffect(1)));

// Activated abilities
this.addAbility(new SimpleActivatedAbility(
    new BoostSourceEffect(2, 2, Duration.EndOfTurn),
    new ManaCostsImpl<>("{1}{R}")));

// Loyalty abilities (planeswalkers)
this.addAbility(new LoyaltyAbility(new DestroyTargetEffect(), -3));

// Abilities with watchers
this.addAbility(new SimpleStaticAbility(effect), new SomeWatcher());

// Abilities with targets
Ability ability = new LoyaltyAbility(new DamageTargetEffect(3), -1);
ability.addTarget(new TargetCreatureOrPlaneswalker());
this.addAbility(ability);
```

### Ability and Effect Class Hierarchy

```
Ability (interface)
├── StaticAbility          # Continuous effects while on battlefield
├── TriggeredAbilityImpl   # Triggers on game events
├── ActivatedAbility       # Costs to activate
├── LoyaltyAbility         # Planeswalker loyalty costs
└── SpellAbility           # Casting the spell itself

Effect (interface)
├── OneShotEffect          # One-time game actions
├── ContinuousEffect       # Ongoing modifications
├── ReplacementEffect      # Replace game events
└── ConditionalEffect      # Conditional wrappers
```

### Set Definition Structure

```java
public final class SomethingSet extends ExpansionSet {
    private static final SomethingSet instance = new SomethingSet();

    public static SomethingSet getInstance() {
        return instance;
    }

    private SomethingSet() {
        super("Set Name", "CODE", ExpansionSet.buildDate(2024, 1, 1), SetType.EXPANSION);
        this.hasBoosters = true;
        this.numBoosterCommon = 11;
        this.numBoosterUncommon = 3;
        this.numBoosterRare = 1;
        this.ratioBoosterMythic = 8;

        cards.add(new SetCardInfo("Card Name", "001", Rarity.COMMON, mage.cards.c.CardName.class));
        // ... more cards
    }
}
```

### Card Generation Utility

Use the Perl script to generate card class stubs from card data:

```bash
cd Utils
perl gen-card.pl "Card Name"
```

This reads from `mtg-cards-data.txt` and `mtg-sets-data.txt` to generate a properly structured card class file.

## Test Patterns

### Writing Card Tests

Tests use a game simulation framework in `Mage.Tests`. The base class `CardTestPlayerBase` sets up a two-player duel:

```java
public class MyCardTest extends CardTestPlayerBase {

    @Test
    public void testBasicFunctionality() {
        // Setup: add cards to zones
        addCard(Zone.HAND, playerA, "Lightning Bolt", 1);
        addCard(Zone.BATTLEFIELD, playerA, "Mountain", 1);

        // Actions: execute game plays
        castSpell(1, PhaseStep.PRECOMBAT_MAIN, playerA, "Lightning Bolt", playerB);

        // Run the game
        setStrictChooseMode(true);
        setStopAt(1, PhaseStep.END_TURN);
        execute();

        // Assertions
        assertLife(playerB, 17);
        assertGraveyardCount(playerA, "Lightning Bolt", 1);
    }
}
```

**Key test API methods:**
- `addCard(Zone, TestPlayer, String cardName, int count)` - Place cards in zones
- `castSpell(turn, PhaseStep, player, spellName)` - Cast a spell
- `castSpell(turn, PhaseStep, player, spellName, targetName)` - Cast targeting something
- `activateAbility(turn, PhaseStep, player, abilityText)` - Activate an ability
- `addTarget(player, targetName)` - Set target for next targeting choice
- `setChoice(player, choiceName)` - Set a choice for the next prompt
- `attack(turn, player, attackerName)` - Declare attacker
- `block(turn, player, blockerName, attackerName)` - Declare blocker
- `setStopAt(turn, PhaseStep)` - Stop game execution at this point
- `setStrictChooseMode(true)` - Fail on unexpected choices (recommended)
- `execute()` - Run the simulated game
- `assertLife(player, expected)` - Check life total
- `assertPermanentCount(player, cardName, expected)` - Check battlefield
- `assertGraveyardCount(player, cardName, expected)` - Check graveyard
- `assertHandCount(player, cardName, expected)` - Check hand
- `assertExileCount(player, cardName, expected)` - Check exile

### Test Locations

```
Mage.Tests/src/test/java/org/mage/test/
├── cards/                    # Tests for specific card mechanics
├── multiplayer/              # Multiplayer-specific tests
├── turnmod/                  # Turn modification tests (extra turns, skip turns)
├── commander/                # Commander format tests
└── ...                       # Additional test categories
```

## Coding Conventions

### General Rules
- **Java 8 compatibility required** - do not use features from Java 9+
- **UTF-8 encoding** for all source files
- All card classes must be declared `public final`
- Card classes use the singleton pattern for keyword abilities (`FlyingAbility.getInstance()`)
- Copy constructors are always `private`
- Complex cards use inner classes for custom abilities, effects, and conditions

### Naming Conventions
- **Modules**: `Mage.ModuleName` (PascalCase with dots)
- **Packages**: `mage.lowercase.dotted`
- **Card classes**: CamelCase matching the card name with no spaces or punctuation
  - "Lightning Bolt" -> `LightningBolt`
  - "Emrakul, the Promised End" -> `EmrakulThePromisedEnd`
  - Card classes go in directory matching first letter: `mage/cards/l/LightningBolt.java`
- **Set classes**: CamelCase matching the set name: `FifthEdition.java`, `ModernHorizons3.java`
- **Test classes**: Suffixed with `Test`: `ExtraTurnsTest.java`

### Mana Cost Strings
Mana costs use brace notation: `"{2}{W}{U}"`, `"{X}{R}{R}"`, `"{B/G}"` (hybrid)

### Card Type Constants
Use `CardType.CREATURE`, `CardType.INSTANT`, `CardType.SORCERY`, `CardType.ENCHANTMENT`, `CardType.ARTIFACT`, `CardType.PLANESWALKER`, `CardType.LAND`, `CardType.BATTLE`

### Supertype and Subtype Constants
- Supertypes: `SuperType.LEGENDARY`, `SuperType.BASIC`, `SuperType.SNOW`
- Subtypes: `SubType.HUMAN`, `SubType.WARRIOR`, `SubType.AURA`, etc.

## CI/CD

### Travis CI
- JDK: OpenJDK 17
- Runs `mvn test -B` on every push
- 2GB JVM heap allocated
- Maven repository cached at `~/.m2`

### GitHub Actions
- **mtg-fetch-cards.yml** - Fetches MTG card references from issues/PRs
- **labeler-auto.yml** / **labeler-manual.yml** - Automatic and manual PR labeling
- **Dependabot** - Weekly dependency update checks for Maven and GitHub Actions

### Code Quality
- **SonarCloud** integration at `sonarcloud.io/project/overview?id=magefree_mage`
- **JaCoCo** code coverage reports generated to `Mage.Reports/target/site/jacoco-aggregate/`

## Plugin Architecture

### Game Format Plugins (`Mage.Game.*`)
Each format plugin provides:
- A `Game` class extending `GameImpl` with format-specific rules
- A `MatchType` class defining player limits and format options
- A `Match` class managing best-of-N series

### Player Plugins (`Mage.Player.*`)
- `Mage.Player.Human` - Human player via network client
- `Mage.Player.AI` - Base computer player with card evaluation
- `Mage.Player.AI.MA` - "Mad Bot" with multi-threaded game simulation (ComputerPlayer7)
- `Mage.Player.AIMCTS` - Monte Carlo Tree Search AI (experimental)
- `Mage.Player.AI.DraftBot` - Draft-specialized AI

### Deck Validators (`Mage.Deck.*`)
- `Mage.Deck.Constructed` - 52 constructed format validators (Standard, Modern, Commander, etc.)
- `Mage.Deck.Limited` - Limited format (draft/sealed) validation

### Tournament Types (`Mage.Tournament.*`)
- Constructed: Swiss and Elimination
- Booster Draft: Swiss and Elimination (including 60+ cube definitions)
- Sealed: Swiss and Elimination (including Jumpstart variants)

## Common Tasks

### Adding a New Card

1. Create the card class in `Mage.Sets/src/mage/cards/[first-letter]/CardName.java`
2. Follow the standard card class structure (constructor, copy constructor, copy method)
3. Add the card to the appropriate set definition(s) in `Mage.Sets/src/mage/sets/`
4. Write tests in `Mage.Tests/` if the card has complex mechanics

Alternatively, use the card generator:
```bash
cd Utils && perl gen-card.pl "Card Name"
```

### Adding a New Set

1. Create `Mage.Sets/src/mage/sets/SetName.java` extending `ExpansionSet`
2. Use the singleton pattern with `getInstance()`
3. Configure booster structure in the constructor
4. Add `SetCardInfo` entries for each card in the set

### Adding a New Game Format

1. Create a new plugin module under `Mage.Server.Plugins/`
2. Implement the `Game`, `MatchType`, and `Match` classes
3. Add the module to `Mage.Server.Plugins/pom.xml`
4. Register the plugin in server configuration

### Adding a New Deck Format Validator

1. Add a new class in `Mage.Server.Plugins/Mage.Deck.Constructed/src/mage/deck/`
2. Extend `Constructed` (or `AbstractCommander` for commander variants)
3. Implement set legality, banned list, and deck size rules
