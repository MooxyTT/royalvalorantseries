import React, { useState, useEffect, useCallback } from "react";
import { Lock, Unlock, Crown } from "lucide-react";

const STORAGE_KEY = "royal-tournament-v3";
const MOD_PIN = "113734";

const TEAM_IDS = ["t1","t2","t3","t4","t5","t6","t7","t8","t9"];

function defaultData() {
  const teams = {};
  TEAM_IDS.forEach((id, i) => {
    teams[id] = { name: `Équipe ${i + 1}`, logo: "" };
  });
  const poule = {};
  TEAM_IDS.forEach((id) => {
    poule[id] = { w: 0, l: 0, mapsF: 0, mapsA: 0, roundsF: 0, roundsA: 0 };
  });
  const emptyMatch = { scoreA: null, scoreB: null };
  return {
    teams,
    poule,
    playins: {
      matchA: { ...emptyMatch },
      matchB: { ...emptyMatch },
    },
    playoffs: {
      ubsf1: { ...emptyMatch },
      ubsf2: { ...emptyMatch },
      finaleUB: { ...emptyMatch },
      lbTour1: { ...emptyMatch },
      finaleLB: { ...emptyMatch },
      grandeFinale: { ...emptyMatch },
    },
  };
}

function getStandings(data) {
  return TEAM_IDS
    .map((id) => ({ id, ...data.poule[id], name: data.teams[id].name, logo: data.teams[id].logo }))
    .sort((a, b) => {
      if (b.w !== a.w) return b.w - a.w;
      if (b.mapsF !== a.mapsF) return b.mapsF - a.mapsF;
      return (b.roundsF - b.roundsA) - (a.roundsF - a.roundsA);
    });
}

// A match is { teamA, teamB, scoreA, scoreB }. Winner is always derived from scores.
function deriveWinner(m) {
  if (!m || !m.teamA || !m.teamB) return null;
  if (m.scoreA == null || m.scoreB == null) return null;
  if (m.scoreA === m.scoreB) return null;
  return m.scoreA > m.scoreB ? m.teamA : m.teamB;
}
function deriveLoser(m) {
  const w = deriveWinner(m);
  if (!w) return null;
  return w === m.teamA ? m.teamB : m.teamA;
}

export default function RoyalTournamentSite() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [tab, setTab] = useState("poule");
  const [isMod, setIsMod] = useState(false);
  const [pinInput, setPinInput] = useState("");
  const [showPinBox, setShowPinBox] = useState(false);
  const [pinError, setPinError] = useState(false);
  const [saveFlash, setSaveFlash] = useState(false);

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get(STORAGE_KEY, true);
        setData(res ? JSON.parse(res.value) : defaultData());
      } catch {
        const d = defaultData();
        setData(d);
        try {
          await window.storage.set(STORAGE_KEY, JSON.stringify(d), true);
        } catch {}
      }
      setLoading(false);
    })();
  }, []);

  const persist = useCallback(async (next) => {
    setData(next);
    try {
      const res = await window.storage.set(STORAGE_KEY, JSON.stringify(next), true);
      if (res) {
        setSaveFlash(true);
        setTimeout(() => setSaveFlash(false), 900);
      }
    } catch {}
  }, []);

  function tryUnlock() {
    if (pinInput === MOD_PIN) {
      setIsMod(true);
      setShowPinBox(false);
      setPinError(false);
      setPinInput("");
    } else {
      setPinError(true);
    }
  }

  if (loading || !data) {
    return (
      <div style={S.loadingWrap}>
        <style>{FONT_IMPORT}</style>
        <Crown size={40} color="#d4af5a" />
        <div style={{ marginTop: 14, color: "#8f8a7c", fontFamily: "Rajdhani, sans-serif", letterSpacing: "0.1em" }}>
          CHARGEMENT DU TOURNOI…
        </div>
      </div>
    );
  }

  const standings = getStandings(data);

  return (
    <div style={S.page}>
      <style>{FONT_IMPORT}</style>

      <header style={S.header}>
        <div style={S.headerInner}>
          <div style={S.brand}>
            <Crown size={30} color="#d4af5a" />
            <div>
              <div style={S.eyebrow}>Royal Valorant Series</div>
              <div style={S.title}>Suivi du tournoi</div>
            </div>
          </div>

          <div>
            {isMod ? (
              <div style={S.modBadge}>
                <Unlock size={14} />
                Mode modérateur
                <button style={S.modExit} onClick={() => setIsMod(false)}>Quitter</button>
              </div>
            ) : showPinBox ? (
              <div style={S.pinRow}>
                <input
                  type="password"
                  value={pinInput}
                  onChange={(e) => { setPinInput(e.target.value); setPinError(false); }}
                  placeholder="Code modérateur"
                  style={S.pinInput}
                  onKeyDown={(e) => e.key === "Enter" && tryUnlock()}
                  autoFocus
                />
                <button style={S.pinBtn} onClick={tryUnlock}>Valider</button>
                {pinError && <span style={S.pinErr}>Code incorrect</span>}
              </div>
            ) : (
              <button style={S.lockBtn} onClick={() => setShowPinBox(true)}>
                <Lock size={13} /> Espace modérateur
              </button>
            )}
          </div>
        </div>

        <nav style={S.tabs}>
          {[
            ["poule", "1 · Phase de poule"],
            ["playins", "2 · Play-Ins"],
            ["playoffs", "3 · Playoffs"],
          ].map(([key, label]) => (
            <button
              key={key}
              onClick={() => setTab(key)}
              style={{ ...S.tabBtn, ...(tab === key ? S.tabBtnActive : {}) }}
            >
              {label}
            </button>
          ))}
        </nav>
      </header>

      <main style={S.main}>
        {tab === "poule" && (
          <PouleView data={data} standings={standings} isMod={isMod} onSave={persist} />
        )}
        {tab === "playins" && (
          <PlayInsView data={data} standings={standings} isMod={isMod} onSave={persist} />
        )}
        {tab === "playoffs" && (
          <PlayoffsView data={data} standings={standings} isMod={isMod} onSave={persist} />
        )}
      </main>

      <footer style={S.footer}>
        <div style={S.footerDivider} />
        Royal Valorant Series — Tournoi de lancement
        {saveFlash && <span style={S.saveFlash}>· Enregistré</span>}
      </footer>
    </div>
  );
}

/* ---------------- AVATAR ---------------- */

function TeamAvatar({ team, size = 22 }) {
  const [error, setError] = useState(false);
  if (!team) return <div style={{ width: size, height: size, flexShrink: 0 }} />;
  const initials = (team.name || "?").trim().split(/\s+/).map((w) => w[0]).slice(0, 2).join("").toUpperCase();
  if (team.logo && !error) {
    return (
      <img
        src={team.logo}
        onError={() => setError(true)}
        alt=""
        style={{ width: size, height: size, borderRadius: "50%", objectFit: "cover", flexShrink: 0, border: `1px solid ${line}` }}
      />
    );
  }
  return (
    <div style={{
      width: size, height: size, borderRadius: "50%", background: goldDim, color: bg,
      fontSize: size * 0.42, fontWeight: 800, display: "flex", alignItems: "center", justifyContent: "center",
      flexShrink: 0, fontFamily: "'Rajdhani',sans-serif",
    }}>
      {initials}
    </div>
  );
}

/* ---------------- POULE ---------------- */

function PouleView({ data, standings, isMod, onSave }) {
  function updateTeamField(id, field, value) {
    const next = structuredClone(data);
    next.teams[id][field] = value;
    onSave(next);
  }
  function updateStat(id, field, value) {
    const next = structuredClone(data);
    next.poule[id][field] = Math.max(0, parseInt(value) || 0);
    onSave(next);
  }

  return (
    <section>
      <SectionHeader
        eyebrow="Étape 1"
        title="Phase de poule"
        sub="Poule unique — chaque équipe affronte toutes les autres en BO3. Top 6 qualifiées, top 2 exemptes du Tour 1."
      />

      <div style={S.tableWrap}>
        <table style={S.table}>
          <thead>
            <tr>
              <th style={S.th}>#</th>
              <th style={{ ...S.th, textAlign: "left" }}>Équipe</th>
              <th style={S.th}>V</th>
              <th style={S.th}>D</th>
              <th style={S.th}>Maps</th>
              <th style={S.th}>Rounds</th>
              <th style={S.th}>Diff</th>
              <th style={S.th}>Statut</th>
            </tr>
          </thead>
          <tbody>
            {standings.map((t, i) => {
              const rank = i + 1;
              const rowStyle = rank <= 2 ? S.rowBlue : rank <= 6 ? S.rowGreen : S.rowRed;
              const tag =
                rank <= 2 ? { txt: "Bye Tour 1", cls: S.tagBlue }
                : rank <= 6 ? { txt: "Play-Ins", cls: S.tagGreen }
                : { txt: "Éliminée", cls: S.tagRed };
              const diff = t.roundsF - t.roundsA;

              return (
                <tr key={t.id} style={rowStyle}>
                  <td style={{ ...S.td, textAlign: "center", fontWeight: 800 }}>{rank}</td>
                  <td style={{ ...S.td, textAlign: "left" }}>
                    {isMod ? (
                      <div style={{ display: "flex", flexDirection: "column", gap: 4, minWidth: 190 }}>
                        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                          <TeamAvatar team={t} />
                          <input
                            style={S.inlineInput}
                            defaultValue={t.name}
                            onBlur={(e) => updateTeamField(t.id, "name", e.target.value)}
                          />
                        </div>
                        <input
                          style={S.logoInput}
                          placeholder="URL du logo (optionnel)"
                          defaultValue={t.logo}
                          onBlur={(e) => updateTeamField(t.id, "logo", e.target.value)}
                        />
                      </div>
                    ) : (
                      <div style={{ display: "flex", alignItems: "center", gap: 9 }}>
                        <TeamAvatar team={t} />
                        <span style={{ fontWeight: 700 }}>{t.name}</span>
                      </div>
                    )}
                  </td>
                  <NumCell val={t.w} isMod={isMod} onCommit={(v) => updateStat(t.id, "w", v)} />
                  <NumCell val={t.l} isMod={isMod} onCommit={(v) => updateStat(t.id, "l", v)} />
                  <td style={S.td}>
                    {isMod ? (
                      <span style={S.pairInput}>
                        <MiniNum val={t.mapsF} onCommit={(v) => updateStat(t.id, "mapsF", v)} />
                        <span style={{ color: inkDim }}>-</span>
                        <MiniNum val={t.mapsA} onCommit={(v) => updateStat(t.id, "mapsA", v)} />
                      </span>
                    ) : (`${t.mapsF}-${t.mapsA}`)}
                  </td>
                  <td style={S.td}>
                    {isMod ? (
                      <span style={S.pairInput}>
                        <MiniNum val={t.roundsF} onCommit={(v) => updateStat(t.id, "roundsF", v)} />
                        <span style={{ color: inkDim }}>-</span>
                        <MiniNum val={t.roundsA} onCommit={(v) => updateStat(t.id, "roundsA", v)} />
                      </span>
                    ) : (`${t.roundsF}-${t.roundsA}`)}
                  </td>
                  <td style={{ ...S.td, fontStyle: "italic", color: diff >= 0 ? "#7fd394" : "#e08a7d" }}>
                    {diff >= 0 ? `+${diff}` : diff}
                  </td>
                  <td style={S.td}><span style={tag.cls}>{tag.txt}</span></td>
                </tr>
              );
            })}
          </tbody>
        </table>
      </div>

      <Legend />
    </section>
  );
}

function NumCell({ val, isMod, onCommit }) {
  return <td style={S.td}>{isMod ? <MiniNum val={val} onCommit={onCommit} /> : val}</td>;
}
function MiniNum({ val, onCommit }) {
  return (
    <input type="number" style={S.miniInput} defaultValue={val} onBlur={(e) => onCommit(e.target.value)} />
  );
}

function Legend() {
  return (
    <div style={S.legend}>
      <span><i style={{ ...S.dot, background: blue }} />Qualifiée directe (bye Tour 1)</span>
      <span><i style={{ ...S.dot, background: green }} />Qualifiée via Play-Ins</span>
      <span><i style={{ ...S.dot, background: red }} />Éliminée</span>
    </div>
  );
}

/* ---------------- PLAY-INS ---------------- */

function PlayInsView({ data, standings, isMod, onSave }) {
  const r = (i) => standings[i];
  const teamA1 = r(2), teamA2 = r(5);
  const teamB1 = r(3), teamB2 = r(4);

  function setScore(matchKey, side, value) {
    const next = structuredClone(data);
    next.playins[matchKey][side] = value === "" ? null : Math.max(0, parseInt(value) || 0);
    onSave(next);
  }

  const matchA = { teamA: teamA1?.id, teamB: teamA2?.id, ...data.playins.matchA };
  const matchB = { teamA: teamB1?.id, teamB: teamB2?.id, ...data.playins.matchB };
  const winA = deriveWinner(matchA);
  const winB = deriveWinner(matchB);

  return (
    <section>
      <SectionHeader
        eyebrow="Étape 2"
        title="Play-Ins — élimination directe"
        sub="Les équipes classées 3 à 6 en poule s'affrontent en BO3. Une seule défaite élimine. Le score entré désigne automatiquement la qualifiée."
      />

      <div style={S.playinStage}>
        <PlayInBracketMatch label="Match A · 3e vs 6e" teamA={teamA1} teamB={teamA2} m={matchA} winner={winA} isMod={isMod} onScore={(side, v) => setScore("matchA", side, v)} top />
        <PlayInBracketMatch label="Match B · 4e vs 5e" teamA={teamB1} teamB={teamB2} m={matchB} winner={winB} isMod={isMod} onScore={(side, v) => setScore("matchB", side, v)} />

        <div style={S.playinConverge}>
          <div style={S.playinConvergeLine} />
          <div style={S.playinQualBox}>
            <div style={S.playinQualLabel}>Qualifiées pour les Playoffs</div>
            <QualRow team={winA ? data.teams[winA] : null} />
            <QualRow team={winB ? data.teams[winB] : null} />
          </div>
        </div>
      </div>

      <div style={S.waitingPanel}>
        Les deux vainqueurs rejoignent <b>{standings[0]?.name}</b> et <b>{standings[1]?.name}</b> en demi-finales des playoffs.
      </div>
    </section>
  );
}

function QualRow({ team }) {
  return (
    <div style={{ display: "flex", alignItems: "center", gap: 8, padding: "6px 0" }}>
      <TeamAvatar team={team} size={18} />
      <span style={{ fontSize: 12.5, color: team ? ink : inkDim, fontStyle: team ? "normal" : "italic" }}>
        {team ? team.name : "En attente"}
      </span>
    </div>
  );
}

function PlayInBracketMatch({ label, teamA, teamB, m, winner, isMod, onScore, top }) {
  const bothKnown = teamA && teamB;
  return (
    <div style={{ ...S.matchCard, marginBottom: top ? 18 : 0 }}>
      <div style={S.matchLabel}>{label}</div>
      <TeamRow team={teamA} score={m.scoreA} isWinner={winner === teamA?.id} isLoser={winner && winner !== teamA?.id}
        editable={isMod && bothKnown} onScoreChange={(v) => onScore("scoreA", v)} />
      <TeamRow team={teamB} score={m.scoreB} isWinner={winner === teamB?.id} isLoser={winner && winner !== teamB?.id}
        editable={isMod && bothKnown} onScoreChange={(v) => onScore("scoreB", v)} />
    </div>
  );
}

function TeamRow({ team, score, isWinner, isLoser, editable, onScoreChange }) {
  return (
    <div style={{
      ...S.teamRow,
      ...(isWinner ? S.teamRowWinner : {}), ...(isLoser ? S.teamRowLoser : {}),
    }}>
      <span style={{ display: "flex", alignItems: "center", gap: 9 }}>
        <TeamAvatar team={team} size={20} />
        {team?.name || "—"}
      </span>
      <span style={{ display: "flex", alignItems: "center", gap: 8 }}>
        {editable ? (
          <input type="number" min="0" style={S.scoreBox} defaultValue={score ?? ""}
            onBlur={(e) => onScoreChange(e.target.value)} />
        ) : (
          score != null && <span style={S.scoreBadge}>{score}</span>
        )}
        {isWinner && <span style={S.winTag}>Qualifiée</span>}
      </span>
    </div>
  );
}

/* ---------------- PLAYOFFS (bracket unifié) ---------------- */

const POS = {
  ubsf1:       { left: 40,  top: 100, width: 250, height: 80 },
  ubsf2:       { left: 40,  top: 260, width: 250, height: 80 },
  finaleUB:    { left: 380, top: 180, width: 250, height: 80 },
  lbTour1:     { left: 40,  top: 460, width: 250, height: 80 },
  finaleLB:    { left: 380, top: 460, width: 250, height: 80 },
  grandeFinale:{ left: 720, top: 320, width: 270, height: 80 },
};

function PlayoffsView({ data, standings, isMod, onSave }) {
  const teams = data.teams;
  const rank1 = standings[0], rank2 = standings[1];

  const pinMatchA = { teamA: standings[2]?.id, teamB: standings[5]?.id, ...data.playins.matchA };
  const pinMatchB = { teamA: standings[3]?.id, teamB: standings[4]?.id, ...data.playins.matchB };
  const winA = deriveWinner(pinMatchA);
  const winB = deriveWinner(pinMatchB);

  const ubsf1 = { teamA: rank1?.id || null, teamB: winB || null, ...data.playoffs.ubsf1 };
  const ubsf2 = { teamA: rank2?.id || null, teamB: winA || null, ...data.playoffs.ubsf2 };
  const wUbsf1 = deriveWinner(ubsf1), wUbsf2 = deriveWinner(ubsf2);

  const finaleUB = { teamA: wUbsf1, teamB: wUbsf2, ...data.playoffs.finaleUB };
  const lbTour1 = { teamA: deriveLoser(ubsf1), teamB: deriveLoser(ubsf2), ...data.playoffs.lbTour1 };
  const wFinaleUB = deriveWinner(finaleUB);
  const wLbTour1 = deriveWinner(lbTour1);

  const finaleLB = { teamA: wLbTour1, teamB: deriveLoser(finaleUB), ...data.playoffs.finaleLB };
  const wFinaleLB = deriveWinner(finaleLB);

  const grandeFinale = { teamA: wFinaleUB, teamB: wFinaleLB, ...data.playoffs.grandeFinale };
  const champion = deriveWinner(grandeFinale);

  function setScore(key, side, value) {
    const next = structuredClone(data);
    next.playoffs[key][side] = value === "" ? null : Math.max(0, parseInt(value) || 0);
    onSave(next);
  }

  const teamOf = (id) => (id ? teams[id] : null);

  return (
    <section>
      <SectionHeader
        eyebrow="Étape 3"
        title="Playoffs — double élimination"
        sub={`${teamOf(rank1?.id)?.name || "Équipe 1"} & ${teamOf(rank2?.id)?.name || "Équipe 2"} (qualifiées directes) + les 2 vainqueurs des Play-Ins. Entrez le score : la qualifiée avance automatiquement.`}
      />

      <div style={S.stageScroll}>
        <div style={S.stage}>
          <svg viewBox="0 0 1010 580" style={S.stageLines}>
            <path d="M290,140 H335 V220 H380" stroke={line} strokeWidth="1.6" fill="none" />
            <path d="M290,300 H335 V220 H380" stroke={line} strokeWidth="1.6" fill="none" />
            <path d="M630,220 H675 V340 H720" stroke={line} strokeWidth="1.6" fill="none" />
            <path d="M290,500 H380" stroke={line} strokeWidth="1.6" fill="none" />
            <path d="M630,500 H675 V380 H720" stroke={line} strokeWidth="1.6" fill="none" />
            <path d="M130,180 V460" stroke={lineDash} strokeWidth="1.4" strokeDasharray="4 4" fill="none" />
            <path d="M200,340 V460" stroke={lineDash} strokeWidth="1.4" strokeDasharray="4 4" fill="none" />
            <path d="M505,260 V460" stroke={lineDash} strokeWidth="1.4" strokeDasharray="4 4" fill="none" />
          </svg>

          <div style={{ ...S.bracketLabelWb, left: 40, top: 30 }}>Winners</div>
          <div style={{ ...S.bracketLabelLb, left: 40, top: 428 }}>Losers</div>

          <RoundTitle x={40} y={72} w={250} text="Demi-finales UB" />
          <RoundTitle x={380} y={152} w={250} text="Finale UB" />
          <RoundTitle x={720} y={292} w={270} text="Grande Finale" />
          <RoundTitle x={40} y={432} w={250} text="Demi-finale LB" />
          <RoundTitle x={380} y={432} w={250} text="Finale LB" />

          <AbsMatch pos={POS.ubsf1} m={ubsf1} winner={wUbsf1} teams={teams} isMod={isMod} onScore={(side, v) => setScore("ubsf1", side, v)} dropTag />
          <AbsMatch pos={POS.ubsf2} m={ubsf2} winner={wUbsf2} teams={teams} isMod={isMod} onScore={(side, v) => setScore("ubsf2", side, v)} dropTag />
          <AbsMatch pos={POS.
