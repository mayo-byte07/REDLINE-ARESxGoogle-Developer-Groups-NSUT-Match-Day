"""REDLINE AI v4 — defensive, observation-driven Pokémon battle agent.

Self-contained (Python standard library only). Implements the arena's expected
PokemonAgent.decide(observation) -> action-string interface.

Important: this is a heuristic agent, not a trained model. It is designed to
fail safely on incomplete observations and never assumes hidden abilities or
unseen move data when a safer fallback is available.
"""

from __future__ import annotations

import math
from typing import Any, Dict, Iterable, List, Optional, Sequence, Tuple


# Attack type -> (super-effective defender types, resisted defender types,
#                 immune defender types). Generation 6+ type chart.
_TYPE_CHART = {
    "normal":   (set(), {"rock", "steel"}, {"ghost"}),
    "fire":     ({"grass", "ice", "bug", "steel"}, {"fire", "water", "rock", "dragon"}, set()),
    "water":    ({"fire", "ground", "rock"}, {"water", "grass", "dragon"}, set()),
    "electric": ({"water", "flying"}, {"electric", "grass", "dragon"}, {"ground"}),
    "grass":    ({"water", "ground", "rock"}, {"fire", "grass", "poison", "flying", "bug", "dragon", "steel"}, set()),
    "ice":      ({"grass", "ground", "flying", "dragon"}, {"fire", "water", "ice", "steel"}, set()),
    "fighting": ({"normal", "ice", "rock", "dark", "steel"}, {"poison", "flying", "psychic", "bug", "fairy"}, {"ghost"}),
    "poison":   ({"grass", "fairy"}, {"poison", "ground", "rock", "ghost"}, {"steel"}),
    "ground":   ({"fire", "electric", "poison", "rock", "steel"}, {"grass", "bug"}, {"flying"}),
    "flying":   ({"grass", "fighting", "bug"}, {"electric", "rock", "steel"}, set()),
    "psychic":  ({"fighting", "poison"}, {"psychic", "steel"}, {"dark"}),
    "bug":      ({"grass", "psychic", "dark"}, {"fire", "fighting", "poison", "flying", "ghost", "steel", "fairy"}, set()),
    "rock":     ({"fire", "ice", "flying", "bug"}, {"fighting", "ground", "steel"}, set()),
    "ghost":    ({"psychic", "ghost"}, {"dark"}, {"normal"}),
    "dragon":   ({"dragon"}, {"steel"}, {"fairy"}),
    "dark":     ({"psychic", "ghost"}, {"fighting", "dark", "fairy"}, set()),
    "steel":    ({"ice", "rock", "fairy"}, {"fire", "water", "electric", "steel"}, set()),
    "fairy":    ({"fighting", "dragon", "dark"}, {"fire", "poison", "steel"}, set()),
}

# Fallback metadata for roster entries visible in the arena. Speeds are base
# species speeds, not exact in-battle stats; use observed stats whenever present.
_SPECIES_DATA = {
    "garchomp":      ({"dragon", "ground"}, 102),
    "dragonite":     ({"dragon", "flying"}, 80),
    "tyranitar":     ({"rock", "dark"}, 61),
    "metagross":     ({"steel", "psychic"}, 70),
    "salamence":     ({"dragon", "flying"}, 100),
    "hydreigon":     ({"dark", "dragon"}, 98),
    "dragapult":     ({"dragon", "ghost"}, 142),
    "volcarona":     ({"bug", "fire"}, 100),
    "lucario":       ({"fighting", "steel"}, 90),
    "scizor":        ({"bug", "steel"}, 65),
    "gengar":        ({"ghost", "poison"}, 110),
    "gyarados":      ({"water", "flying"}, 81),
    "charizard":     ({"fire", "flying"}, 100),
    "blastoise":     ({"water"}, 78),
    "venusaur":      ({"grass", "poison"}, 80),
    "arcanine":      ({"fire"}, 95),
    "excadrill":     ({"ground", "steel"}, 88),
    "togekiss":      ({"fairy", "flying"}, 80),
    "snorlax":       ({"normal"}, 30),
    "alakazam":      ({"psychic"}, 120),
    "lapras":        ({"water", "ice"}, 60),
    "corviknight":   ({"flying", "steel"}, 67),
    "ferrothorn":    ({"grass", "steel"}, 20),
    "greninja":      ({"water", "dark"}, 122),
}

_PRIORITY_MOVES = {
    "accelerock", "aquajet", "bulletpunch", "extremespeed", "fakeout",
    "iceshard", "machpunch", "quickattack", "shadowsneak", "suckerpunch",
    "vacuumwave", "watershuriken", "grassyglide", "firstimpression",
}
_SETUP_BOOSTS = {
    "swordsdance": {"atk": 2},
    "nastyplot": {"spa": 2},
    "quiverdance": {"spa": 1, "spd": 1, "spe": 1},
    "dragondance": {"atk": 1, "spe": 1},
    "calmmind": {"spa": 1, "spd": 1},
    "bulkup": {"atk": 1, "def": 1},
    "shellsmash": {"atk": 2, "spa": 2, "spe": 2, "def": -1, "spd": -1},
    "agility": {"spe": 2},
    "rockpolish": {"spe": 2},
    "irondefense": {"def": 2},
    "amnesia": {"spd": 2},
    "honeclaws": {"atk": 1, "accuracy": 1},
    "tailglow": {"spa": 3},
    "shiftgear": {"atk": 1, "spe": 2},
}
_RECOVERY_MOVES = {
    "recover", "roost", "softboiled", "slackoff", "synthesis", "morningsun",
    "shoreup", "milkdrink", "wish", "rest", "recovering", "strengthsap",
}
_STATUS_MOVES = {
    "sleeppowder", "spore", "hypnosis", "darkvoid", "thunderwave", "willowisp",
    "toxic", "stunspore", "glare", "yawn", "nuzzle", "leechseed",
}
_SWITCH_MOVES = {"uturn", "voltswitch", "flipturn", "partingshot"}
_SELF_DROP_MOVES = {"closecombat", "dracometeor", "overheat", "superpower", "leafstorm", "makeitrain"}
_LOCKING_MOVES = {"outrage", "thrash", "petaldance"}


def _dict(value: Any) -> Dict[str, Any]:
    return value if isinstance(value, dict) else {}


def _list(value: Any) -> List[Any]:
    if isinstance(value, list):
        return value
    if isinstance(value, tuple):
        return list(value)
    return []


def _num(value: Any, default: float = 0.0) -> float:
    try:
        # Booleans are not meaningful numeric observations here.
        if isinstance(value, bool) or value is None:
            return default
        result = float(value)
        return result if math.isfinite(result) else default
    except (TypeError, ValueError, OverflowError):
        return default


def _norm(value: Any) -> str:
    if isinstance(value, dict):
        value = value.get("id") or value.get("name") or value.get("value") or ""
    return "".join(ch for ch in str(value or "").strip().lower() if ch.isalnum())


def _clamp(value: float, low: float, high: float) -> float:
    return max(low, min(high, value))


def _stage_mult(stage: Any) -> float:
    stage_value = int(_clamp(_num(stage), -6, 6))
    return (2.0 + stage_value) / 2.0 if stage_value >= 0 else 2.0 / (2.0 - stage_value)


def _species(pokemon: Dict[str, Any]) -> str:
    return _norm(pokemon.get("species") or pokemon.get("name") or pokemon.get("id"))


def _get_types(pokemon: Any) -> List[str]:
    poke = _dict(pokemon)
    raw = poke.get("types") or poke.get("type")
    if isinstance(raw, str):
        # Accept a single type or strings such as "water/flying".
        types = [_norm(part) for part in raw.replace(",", "/").split("/") if _norm(part)]
        if types:
            return types
    elif isinstance(raw, (list, tuple, set)):
        types = []
        for entry in raw:
            type_name = _norm(entry)
            if type_name:
                types.append(type_name)
        if types:
            return types
    meta = _SPECIES_DATA.get(_species(poke))
    return sorted(meta[0]) if meta else []


def _get_ability(pokemon: Any) -> str:
    poke = _dict(pokemon)
    # Do not guess abilities from species: many roster Pokémon have multiple
    # legal abilities, and a guessed immunity can make the agent misplay.
    return _norm(poke.get("ability") or poke.get("active_ability") or poke.get("base_ability"))


def _get_item(pokemon: Any) -> str:
    poke = _dict(pokemon)
    return _norm(poke.get("item") or poke.get("held_item") or poke.get("heldItem"))


def _get_status(pokemon: Any) -> str:
    return _norm(_dict(pokemon).get("status"))


def _get_boosts(pokemon: Any) -> Dict[str, Any]:
    return _dict(_dict(pokemon).get("boosts"))


def _estimate_speed(pokemon: Any) -> float:
    poke = _dict(pokemon)
    stats = _dict(poke.get("stats"))
    raw_speed = 0.0
    for key in ("spe", "speed", "speed_stat"):
        raw_speed = _num(poke.get(key), 0.0)
        if raw_speed > 0:
            break
    if raw_speed <= 0:
        raw_speed = _num(stats.get("spe") or stats.get("speed"), 0.0)
    if raw_speed <= 0:
        raw_speed = float(_SPECIES_DATA.get(_species(poke), (set(), 80))[1])
    raw_speed *= _stage_mult(_get_boosts(poke).get("spe", 0))
    if _get_item(poke) == "choicescarf":
        raw_speed *= 1.5
    if _get_status(poke) in {"par", "paralysis", "paralyzed", "paralysed"}:
        raw_speed *= 0.5
    conditions = _dict(poke.get("side_conditions"))
    if conditions.get("tailwind"):
        raw_speed *= 2.0
    return max(1.0, raw_speed)


def _ability_immunity_multiplier(attack_type: str, defender_ability: str, attacker_ability: str) -> Optional[float]:
    bypass = attacker_ability in {"moldbreaker", "teravolt", "turboblaze"}
    if not bypass:
        if attack_type == "ground" and defender_ability == "levitate":
            return 0.0
        if attack_type == "fire" and defender_ability == "flashfire":
            return 0.0
        if attack_type == "water" and defender_ability in {"waterabsorb", "stormdrain", "dryskin"}:
            return 0.0
        if attack_type == "electric" and defender_ability in {"voltabsorb", "motordrive", "lightningrod"}:
            return 0.0
        if attack_type == "grass" and defender_ability == "sapsipper":
            return 0.0
    return None


def _type_mult(attack_type: Any, defender_types: Any, defender_ability: Any = "", attacker_ability: Any = "") -> float:
    attack = _norm(attack_type)
    if not attack or attack not in _TYPE_CHART:
        return 1.0
    def_types: List[str]
    if isinstance(defender_types, str):
        def_types = [_norm(part) for part in defender_types.replace(",", "/").split("/") if _norm(part)]
    else:
        def_types = [_norm(t) for t in _list(defender_types) if _norm(t)]
    defender_ability = _norm(defender_ability)
    attacker_ability = _norm(attacker_ability)
    ability_mult = _ability_immunity_multiplier(attack, defender_ability, attacker_ability)
    if ability_mult is not None:
        return ability_mult

    super_types, resisted_types, immune_types = _TYPE_CHART[attack]
    multiplier = 1.0
    for defender_type in def_types:
        if defender_type in immune_types:
            scrappy_bypass = (defender_type == "ghost" and attack in {"normal", "fighting"}
                              and attacker_ability in {"scrappy", "mindseye"})
            if scrappy_bypass:
                continue
            return 0.0
        if defender_type in super_types:
            multiplier *= 2.0
        elif defender_type in resisted_types:
            multiplier *= 0.5
    if not (attacker_ability in {"moldbreaker", "teravolt", "turboblaze"}):
        if defender_ability == "thickfat" and attack in {"fire", "ice"}:
            multiplier *= 0.5
        elif defender_ability == "heatproof" and attack == "fire":
            multiplier *= 0.5
        elif defender_ability == "waterbubble" and attack == "fire":
            multiplier *= 0.5
    return multiplier


def _move_type(move: Any) -> str:
    return _norm(_dict(move).get("type") or _dict(move).get("move_type"))


def _move_category(move: Any) -> str:
    return _norm(_dict(move).get("category") or _dict(move).get("damage_class"))


def _move_power(move: Any) -> float:
    data = _dict(move)
    for key in ("base_power", "power", "bp"):
        if key in data and data[key] is not None:
            return max(0.0, _num(data[key], 0.0))
    return 0.0


def _move_accuracy(move: Any) -> float:
    raw = _dict(move).get("accuracy", 100)
    if isinstance(raw, bool) or raw is None:
        return 1.0
    accuracy = _num(raw, 100.0)
    # Accept both Showdown-style percentages (90) and ratio values (0.9).
    if 0.0 < accuracy <= 1.0:
        return _clamp(accuracy, 0.0, 1.0)
    return _clamp(accuracy / 100.0, 0.0, 1.0)


def _get_move_id(move: Any) -> str:
    data = _dict(move)
    return _norm(data.get("id") or data.get("name") or data.get("move"))


def _get_hp(pokemon: Any, default: float = 100.0) -> float:
    poke = _dict(pokemon)
    for key in ("hp_percent", "hp_pct", "health_percent", "percent_hp"):
        if key in poke:
            return _clamp(_num(poke.get(key), default), 0.0, 100.0)
    hp = _num(poke.get("hp"), -1.0)
    max_hp = _num(poke.get("max_hp") or poke.get("maxhp"), 0.0)
    if hp >= 0 and max_hp > 0:
        return _clamp(100.0 * hp / max_hp, 0.0, 100.0)
    return _clamp(default, 0.0, 100.0)


def _known_foe_moves(foe: Dict[str, Any]) -> List[Dict[str, Any]]:
    raw = foe.get("known_moves") or foe.get("moves") or foe.get("revealed_moves") or []
    result = []
    for move in _list(raw):
        if isinstance(move, dict):
            result.append(move)
    return result


def _worst_incoming_mult(foe: Dict[str, Any], own: Dict[str, Any]) -> float:
    foe_moves = _known_foe_moves(foe)
    own_types = _get_types(own)
    own_ability = _get_ability(own)
    foe_ability = _get_ability(foe)
    move_mults = []
    for move in foe_moves:
        move_type = _move_type(move)
        if not move_type or _move_category(move) == "status":
            continue
        # Type is enough for defensive matchup evaluation. Some arena versions
        # omit base_power from revealed moves, so don't discard those records.
        move_mults.append(_type_mult(move_type, own_types, own_ability, foe_ability))
    if move_mults:
        return max(move_mults)

    # Unknown moves: opposing STAB types are only a fallback estimate, not facts
    # about its actual moveset.
    foe_types = _get_types(foe)
    if foe_types:
        return max(_type_mult(t, own_types, own_ability, foe_ability) for t in foe_types)
    return 1.0


def _best_stab_multiplier(attacker: Dict[str, Any], defender: Dict[str, Any]) -> float:
    atk_types = _get_types(attacker)
    defender_types = _get_types(defender)
    defender_ability = _get_ability(defender)
    attacker_ability = _get_ability(attacker)
    if not atk_types:
        return 1.0
    return max(_type_mult(t, defender_types, defender_ability, attacker_ability) for t in atk_types)


def _stat(pokemon: Dict[str, Any], names: Sequence[str]) -> float:
    stats = _dict(pokemon.get("stats"))
    for name in names:
        value = _num(pokemon.get(name), 0.0)
        if value > 0:
            return value
        value = _num(stats.get(name), 0.0)
        if value > 0:
            return value
    return 0.0


def _attack_defense_factor(active: Dict[str, Any], foe: Dict[str, Any], category: str) -> float:
    if category == "physical":
        attack = _stat(active, ("atk", "attack"))
        defense = _stat(foe, ("def", "defense"))
        attack_stage = _num(_get_boosts(active).get("atk", 0))
        defense_stage = _num(_get_boosts(foe).get("def", 0))
    elif category == "special":
        attack = _stat(active, ("spa", "spatk", "special_attack", "sp_attack"))
        defense = _stat(foe, ("spd", "spdef", "special_defense", "sp_defense"))
        attack_stage = _num(_get_boosts(active).get("spa", 0))
        defense_stage = _num(_get_boosts(foe).get("spd", 0))
    else:
        return 1.0
    if attack <= 0 or defense <= 0:
        return 1.0
    ratio = _clamp(attack / defense, 0.5, 2.0)
    return ratio * _stage_mult(attack_stage) / _stage_mult(defense_stage)


def _is_choice_item(item: str) -> bool:
    return item in {"choiceband", "choicespecs", "choicescarf"}


def _is_fainted(pokemon: Dict[str, Any]) -> bool:
    if bool(pokemon.get("fainted")):
        return True
    # Missing HP telemetry is unknown, not equivalent to a fainted Pokémon.
    for key in ("hp_percent", "hp_pct", "health_percent", "percent_hp"):
        if key in pokemon:
            return _get_hp(pokemon, 0.0) <= 0.0
    hp = _num(pokemon.get("hp"), -1.0)
    max_hp = _num(pokemon.get("max_hp") or pokemon.get("maxhp"), 0.0)
    return hp == 0.0 or (hp >= 0.0 and max_hp > 0.0 and hp <= 0.0)


class PokemonAgent:
    """Observation-driven battle heuristic for a 3v3 Pokémon battle arena."""

    def __init__(self) -> None:
        self.last_turn = -1
        self.last_action: Optional[str] = None

    def _team_preview(self, obs: Dict[str, Any]) -> str:
        # The arena accepts this ordering token; if the observation exposes only
        # a preview flag (as in the current UI), no additional information can
        # support a better ordering.
        return "team 123"

    def _position_score(self, pokemon: Dict[str, Any], obs: Dict[str, Any]) -> float:
        if _is_fainted(pokemon):
            return -10000.0
        foe = _dict(obs.get("foe"))
        hp = _get_hp(pokemon, 100.0)
        incoming = _worst_incoming_mult(foe, pokemon)
        score = hp * 0.45
        if _get_status(pokemon):
            score -= 7.0
        if incoming >= 4.0:
            score -= 48.0
        elif incoming >= 2.0:
            score -= 25.0
        elif incoming == 0.0:
            score += 23.0
        elif incoming <= 0.5:
            score += 13.0
        if _get_ability(pokemon) == "multiscale" and hp >= 99.0 and incoming <= 1.0:
            score += 8.0

        best_offense = _best_stab_multiplier(pokemon, foe)
        if best_offense >= 4.0:
            score += 24.0
        elif best_offense >= 2.0:
            score += 16.0
        elif best_offense == 0.0:
            score -= 14.0
        elif best_offense < 1.0:
            score -= 6.0

        # If a switch entry contains its own moves, give it modest credit for
        # coverage that raw typing would miss.
        candidate_moves = _list(pokemon.get("moves") or pokemon.get("known_moves"))
        candidate_obs = dict(obs)
        candidate_obs["my_active"] = pokemon
        candidate_scores = [self._move_score(m, candidate_obs) for m in candidate_moves if isinstance(m, dict)]
        if candidate_scores:
            score += _clamp(max(candidate_scores) / 30.0, 0.0, 12.0)
        return score

    def _forced_switch(self, switches: List[Dict[str, Any]], obs: Dict[str, Any]) -> str:
        # Keep the original list position for simplified observations that omit
        # explicit switch indices.
        legal = [(s, position) for position, s in enumerate(switches)
                 if not _is_fainted(s) and self._action_index(s, position) is not None]
        if not legal:
            return "pass"
        best, position = max(legal, key=lambda pair: self._position_score(pair[0], obs))
        return "switch {}".format(self._action_index(best, position))

    @staticmethod
    def _observed_multiplier(move: Dict[str, Any], active: Dict[str, Any], foe: Dict[str, Any]) -> float:
        if "type_multiplier" in move and move.get("type_multiplier") is not None:
            multiplier = max(0.0, _num(move.get("type_multiplier"), 1.0))
            # Preserve explicitly known ability immunities in case an upstream
            # field reports a type-only multiplier.
            move_type = _move_type(move)
            ability_mult = _ability_immunity_multiplier(move_type, _get_ability(foe), _get_ability(active))
            if ability_mult == 0.0:
                return 0.0
            return multiplier
        move_type = _move_type(move)
        foe_types = _get_types(foe)
        name = _get_move_id(move)
        mult = _type_mult(move_type, foe_types, _get_ability(foe), _get_ability(active))
        # Move-specific type interactions not represented by the standard chart.
        if name == "freezedry" and "water" in foe_types and mult > 0.0:
            # Freeze-Dry treats each Water typing as weak to Ice instead of
            # resistant; a Water dual-type changes the multiplier by x4.
            mult *= 4.0 ** foe_types.count("water")
        elif name == "thousandarrows" and "flying" in foe_types:
            # Thousand Arrows can hit airborne/Flying targets.
            grounded_types = [t for t in foe_types if t != "flying"]
            mult = _type_mult("ground", grounded_types, "", _get_ability(active))
        return mult

    def _setup_score(self, name: str, move: Dict[str, Any], obs: Dict[str, Any], active: Dict[str, Any], foe: Dict[str, Any], incoming: float) -> float:
        item = _get_item(active)
        if _is_choice_item(item):
            return -35.0
        my_hp = _get_hp(active, 100.0)
        foe_hp = _get_hp(foe, 100.0)
        turn = int(_num(obs.get("turn", 1), 1))
        if my_hp < 62.0 or foe_hp < 28.0 or incoming >= 2.0 or turn > 8:
            return -25.0

        boosts = _get_boosts(active)
        spec = _dict(move.get("boosts"))
        target_boosts = spec if spec else _SETUP_BOOSTS.get(name, {})
        if not target_boosts:
            return -10.0
        remaining = []
        for stat, amount in target_boosts.items():
            if amount > 0 and stat in {"atk", "def", "spa", "spd", "spe", "accuracy", "evasion"}:
                remaining.append(max(0.0, 6.0 - _num(boosts.get(stat, 0))))
        if not remaining or max(remaining) <= 0:
            return -15.0
        return 118.0 if _num(obs.get("turn", 1), 1) <= 3 and my_hp >= 78.0 else 88.0

    def _move_score(self, move: Any, obs: Dict[str, Any]) -> float:
        m = _dict(move)
        active = _dict(obs.get("my_active"))
        foe = _dict(obs.get("foe"))
        if not m:
            return -10000.0

        name = _get_move_id(m)
        move_type = _move_type(m)
        category = _move_category(m)
        power = _move_power(m)
        accuracy_factor = _move_accuracy(m)
        my_hp = _get_hp(active, 100.0)
        foe_hp = _get_hp(foe, 100.0)
        own_types = _get_types(active)
        own_ability = _get_ability(active)
        own_item = _get_item(active)
        boosts = _get_boosts(active)
        incoming = _worst_incoming_mult(foe, active)
        mult = self._observed_multiplier(m, active, foe) if move_type else 1.0
        if power > 0 and mult <= 0.0:
            return -10000.0

        is_stab = bool(m.get("stab")) or (move_type in own_types)
        stab = 1.5 if is_stab else 1.0
        effective_power = power
        foe_item = _get_item(foe)
        foe_status = _get_status(foe)
        if name == "facade" and _get_status(active):
            effective_power *= 2.0
        elif name == "hex" and foe_status:
            effective_power *= 2.0
        elif name == "venoshock" and foe_status in {"psn", "tox", "poison", "toxic"}:
            effective_power *= 2.0
        elif name == "brine" and foe_hp <= 50.0:
            effective_power *= 2.0
        elif name == "knockoff" and foe_item:
            effective_power *= 1.5
        elif name == "acrobatics" and not own_item:
            effective_power *= 2.0

        # Respect actual, observed ability/item only. Unknown ability or item is
        # deliberately not guessed from species.
        if own_ability == "technician" and 0 < effective_power <= 60:
            effective_power *= 1.5
        if own_item == "metalcoat" and move_type == "steel":
            effective_power *= 1.2
        elif own_item == "choiceband" and category == "physical":
            effective_power *= 1.5
        elif own_item == "choicespecs" and category == "special":
            effective_power *= 1.5
        elif own_item == "lifeorb":
            effective_power *= 1.3
        elif own_item == "expertbelt" and mult > 1.0:
            effective_power *= 1.2
        elif own_item == "muscleband" and category == "physical":
            effective_power *= 1.1
        elif own_item == "wiseglasses" and category == "special":
            effective_power *= 1.1

        if effective_power > 0:
            score = effective_power * accuracy_factor * mult * stab
            score *= _attack_defense_factor(active, foe, category)
        else:
            score = 0.0

        # Burn penalizes physical attackers in most formats unless Guts is known.
        if category == "physical" and _get_status(active) in {"brn", "burn", "burned"} and own_ability != "guts":
            score *= 0.5
        if category == "special" and _get_status(active) in {"par", "paralysis", "paralyzed", "paralysed"}:
            # Paralysis mostly affects speed, but a few formats/abilities vary;
            # don't incorrectly cut special damage.
            pass

        foe_speed = _estimate_speed(foe)
        my_speed = _estimate_speed(active)
        priority_value = _num(m.get("priority"), 0.0)
        if name in _PRIORITY_MOVES:
            priority_value = max(priority_value, 1.0)
        if priority_value > 0:
            if foe_hp <= 30.0:
                score *= 1.45
            elif my_speed < foe_speed and (my_hp <= 45.0 or incoming >= 2.0):
                score *= 1.30
            else:
                score *= 1.08

        # Known damage estimate supplied by the arena takes precedence for KO
        # heuristics, but the generic damage score still handles formats where
        # no estimate is included.
        damage_percent = None
        for key in ("damage_percent", "estimated_damage_percent", "expected_damage_percent", "damage_pct"):
            if m.get(key) is not None:
                damage_percent = _clamp(_num(m.get(key), 0.0), 0.0, 200.0)
                break
        if damage_percent is None:
            direct_damage = _num(m.get("estimated_damage"), -1.0)
            foe_max_hp = _num(foe.get("max_hp") or foe.get("maxhp"), 0.0)
            if direct_damage >= 0 and foe_max_hp > 0:
                damage_percent = 100.0 * direct_damage / foe_max_hp
        if damage_percent is not None and effective_power > 0:
            if damage_percent >= foe_hp:
                score += 160.0
            elif damage_percent >= foe_hp * 0.75:
                score += 55.0

        if effective_power > 0 and foe_hp <= 25.0:
            score *= 1.22
        elif effective_power > 0 and foe_hp <= 40.0:
            score *= 1.10

        if name in _SELF_DROP_MOVES and foe_hp > 20.0:
            score *= 0.88
        if name in _LOCKING_MOVES and foe_hp > 30.0:
            score *= 0.90
        if name in _SWITCH_MOVES:
            if any(_num(boosts.get(stat, 0)) > 0 for stat in ("atk", "spa", "spe")):
                score *= 0.70
            elif incoming >= 2.0:
                score *= 1.08

        # Do not spend a turn on utility with a Choice item; the item may be
        # locked to it and switching is usually the only productive response.
        if category == "status" or effective_power <= 0:
            if name in _SETUP_BOOSTS or _dict(m.get("boosts")):
                return self._setup_score(name, m, obs, active, foe, incoming)

            if name in _RECOVERY_MOVES:
                if _is_choice_item(own_item) or my_hp >= 78.0 or incoming >= 2.0:
                    return -20.0
                recovery_value = (100.0 - my_hp) * (1.65 if my_hp <= 35.0 else 1.35)
                if name == "rest" and my_hp < 35.0:
                    recovery_value += 18.0
                return max(0.0, recovery_value * accuracy_factor)

            foe_status = _get_status(foe)
            if name in _STATUS_MOVES:
                if _is_choice_item(own_item) or foe_status or foe_hp <= 15.0:
                    return -20.0
                if incoming >= 2.0 and my_speed < foe_speed:
                    return -5.0
                value = 72.0
                if name in {"spore", "sleeppowder", "hypnosis", "darkvoid", "yawn"}:
                    value = 92.0 if not foe_status else 10.0
                if name in {"thunderwave", "glare", "stunspore", "nuzzle"} and my_speed < foe_speed:
                    value += 10.0
                if name == "willowisp":
                    foe_physical = any(_move_category(x) == "physical" for x in _known_foe_moves(foe))
                    value += 12.0 if foe_physical else 0.0
                if name == "toxic" and foe_hp > 55.0:
                    value += 8.0
                return value * accuracy_factor

            if name in {"protect", "detect", "kingsshield", "spikyshield"}:
                if my_hp <= 18.0 and foe_hp > 10.0:
                    return 28.0
                # Protect is only a mild option if residual damage/status is
                # likely to finish the foe, otherwise it stalls a useful turn.
                if _get_status(foe) in {"psn", "tox", "poison", "toxic", "brn", "burn"} and my_hp > foe_hp:
                    return 35.0
                return -15.0

            utility_names = {
                "taunt": 45.0, "encore": 58.0, "knockoff": 0.0,
                "reflect": 28.0, "lightscreen": 28.0, "auroraveil": 38.0,
                "stealthrock": 32.0, "spikes": 25.0, "stickyweb": 28.0,
                "rapidspin": 25.0, "defog": 20.0, "trickroom": 15.0,
                "leechseed": 60.0,
            }
            if name in utility_names:
                if _is_choice_item(own_item):
                    return -20.0
                value = utility_names[name]
                if name in {"reflect", "lightscreen", "auroraveil"} and _num(obs.get("turn", 1), 1) <= 3:
                    value += 12.0
                if name in {"stealthrock", "spikes", "stickyweb"} and _num(obs.get("turn", 1), 1) > 4:
                    value -= 10.0
                return value * accuracy_factor

            # Any unrecognized status move should not outrank damaging moves by
            # accident, but retain a tiny score if no attacks are available.
            return 3.0 * accuracy_factor

        # Secondary-effect utility supplied by the observation, when present.
        secondary = _clamp(_num(m.get("secondary_chance"), 0.0), 0.0, 100.0)
        if secondary > 0 and my_speed >= foe_speed:
            score += min(12.0, secondary * 0.10)
        return score

    @staticmethod
    def _action_index(entry: Dict[str, Any], fallback_position: int) -> Any:
        for key in ("index", "move_index", "switch_index", "slot"):
            if entry.get(key) is not None:
                return entry[key]
        # Most arena observations include index explicitly; positional fallback
        # avoids producing "pass" for otherwise valid simplified test fixtures.
        return fallback_position

    def decide(self, observation):
        obs = _dict(observation)
        if obs.get("team_preview"):
            return self._team_preview(obs)

        turn = int(_num(obs.get("turn", self.last_turn + 1), self.last_turn + 1))
        raw_moves = _list(obs.get("valid_moves"))
        # Preserve source positions for observations that do not provide index.
        moves_with_position = [(m, pos) for pos, m in enumerate(raw_moves)
                               if isinstance(m, dict) and not m.get("disabled", False)]
        moves = [m for m, _ in moves_with_position]
        switches = [s for s in _list(obs.get("valid_switches")) if isinstance(s, dict)]

        if obs.get("force_switch"):
            action = self._forced_switch(switches, obs)
            self.last_turn, self.last_action = turn, action
            return action

        if not moves:
            action = self._forced_switch(switches, obs) if switches else "pass"
            self.last_turn, self.last_action = turn, action
            return action

        active = _dict(obs.get("my_active"))
        foe = _dict(obs.get("foe"))
        # If the UI exposes the move lock, restrict to it. Usually the engine
        # already filters valid_moves, so this is a defensive extra check.
        locked_move = _norm(active.get("locked_move") or obs.get("locked_move"))
        if _is_choice_item(_get_item(active)) and locked_move:
            locked = [m for m in moves if _get_move_id(m) == locked_move]
            if locked:
                moves = locked

        eligible_ids = {id(move) for move in moves}
        scored_moves = [(self._move_score(move, obs), position, move)
                        for move, position in moves_with_position if id(move) in eligible_ids]
        best_score, best_position, best_move = max(scored_moves, key=lambda item: item[0])

        # Avoid switching back and forth every turn immediately after a switch.
        just_switched = (
            isinstance(self.last_action, str)
            and self.last_action.startswith("switch ")
            and (turn - self.last_turn) <= 1
        )
        can_switch = [(s, position) for position, s in enumerate(switches)
                      if not _is_fainted(s) and self._action_index(s, position) is not None]

        if can_switch and not just_switched:
            best_switch, switch_position = max(can_switch, key=lambda pair: self._position_score(pair[0], obs))
            switch_score = self._position_score(best_switch, obs)
            current_score = self._position_score(active, obs) if active else 0.0
            incoming = _worst_incoming_mult(foe, active)
            switch_hp = _get_hp(best_switch, 0.0)
            current_hp = _get_hp(active, 100.0)
            slower = _estimate_speed(active) < _estimate_speed(foe)

            # A switch must have a meaningful positional advantage, or rescue a
            # nearly-fainted Pokémon from a likely poor trade. Do not switch out
            # when a calculated KO is explicitly exposed by the arena.
            estimated_ko = False
            damage_pct = _num(best_move.get("damage_percent") or best_move.get("estimated_damage_percent"), -1.0)
            if damage_pct >= _get_hp(foe, 100.0) and _move_power(best_move) > 0:
                estimated_ko = True

            severe_danger = incoming >= 4.0 and switch_score >= current_score - 2.0 and best_score < 220.0
            bad_matchup = incoming >= 2.0 and switch_score >= current_score + 10.0 and best_score < 145.0
            tactical_counter = incoming >= 1.0 and switch_score >= current_score + 18.0 and best_score < 125.0
            low_hp_save = current_hp <= 22.0 and slower and switch_hp >= 35.0 and switch_score >= current_score - 5.0 and best_score < 180.0
            if not estimated_ko and (severe_danger or bad_matchup or tactical_counter or low_hp_save):
                action = "switch {}".format(self._action_index(best_switch, switch_position))
                self.last_turn, self.last_action = turn, action
                return action

        move_index = self._action_index(best_move, best_position)
        action = "move {}".format(move_index)
        self.last_turn, self.last_action = turn, action
        return action


# Alias kept for harnesses that import a generic Agent name.
Agent = PokemonAgent
