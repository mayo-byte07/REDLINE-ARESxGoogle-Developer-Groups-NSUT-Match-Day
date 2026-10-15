REDLINE AI v4 Competitive
A self-contained Python heuristic agent for the REDLINE / PokéDuel AI Pokémon battle arena. The agent evaluates the current battle observation and returns a move or switch action through the required PokemonAgent.decide(self, observation) interface.
Important: This is a heuristic decision agent, not a trained neural network or a guaranteed winner. Its actual win rate must be measured by running battles in the arena.

Features
- Move scoring: weighs base power, accuracy, STAB, type effectiveness, move category, available stat information, status, boosts, and priority where those fields are provided.
- Defensive matchup checks: considers Pokémon types, known moves, relevant immunities, status, and remaining HP.
- Tactical actions: evaluates setup, recovery, status moves, and priority attacks using condition-based heuristics.
- Switch decisions: compares the active Pokémon with available switch targets and includes safeguards intended to reduce repetitive switching.
- Robust input handling: normalizes incomplete observations and uses safe fallbacks when optional fields are missing.
- No third-party dependencies: uses Python's standard library only.
Files
File	Purpose
redline_ai.py	Python agent submitted to the arena
team.txt	Proposed three-Pokémon team in Showdown-style text format
TEAM.md	Team composition and intended roles
requirements.txt	Dependency declaration (standard library only)


Arena compatibility
The arena documentation describes a Gen 9, 3v3 Singles format and requires a Python file defining PokemonAgent with decide(self, observation). This repository follows that interface and returns action strings such as:
- team 123 during team preview
- move 1 through move 4 for a legal move slot
- switch 2 / switch 3 for a legal bench slot
- pass when no move or switch is available in the supplied observation
The arena's live validator is authoritative. Always use Show Error in the portal if the service reports an interface or action error.
Use it in the arena
1. Open the REDLINE Pokémon AI Battle Arena.
2. In Customize Team, select the three Pokémon listed in team.txt and configure the listed abilities, items, natures, EVs, and moves where the portal permits them.
3. Upload redline_ai.py under Player 1 or Player 2.
4. Click Start Simulation.
5. Use Show Winner to inspect the outcome, and Show Error if a run fails.
6. Repeat against different opponents and record all outcomes before calculating a win rate.
Team legality note
The arena uses a curated roster and move pool. team.txt is a suggested team configuration in a familiar Showdown-style format; it is not a claim that the web UI imports that file automatically. Before submission, verify each move/item/ability in Customize Team and adjust the text file if the official portal or event rules reject an option.
Run a local smoke test
Python 3.9+ is recommended.
python -m py_compile redline_ai.py
python - <<'PY'
from redline_ai import PokemonAgent
agent = PokemonAgent()
print(agent.decide({"team_preview": True}))
print(agent.decide({"valid_moves": [], "valid_switches": []}))
PY
Expected output:
team 123
pass
These checks verify import/interface basics only. They do not measure competitive win rate or reproduce the full battle engine.
Testing and reporting results
Use a fixed own team and test against varied opponent teams/agents. Save each battle's winner, opponent, and any error. For W wins out of N completed battles:
Win rate = (W / N) * 100%
Do not treat a small number of matches as a reliable estimate of long-run performance.
Limitations
- Decision quality depends on the fields exposed in each observation.
- Team preview currently returns the default team 123 ordering because the current interface does not provide enough preview information in the code to select a matchup-specific order.
- Heuristic scores approximate tactical value; they are not a full Pokémon damage calculator and do not model every battle mechanic.
- No live 10-match benchmark is included in this repository.
References
- Battle Arena
- Arena documentation
- ARES × Shunya event listing and submission overview
