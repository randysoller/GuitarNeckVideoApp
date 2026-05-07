# GuitarNeckVideoApp
import React, { useState, useRef } from "react";
import { AlphaTabApi } from "@coderline/alphatab";

/* =========================
   CONFIG
========================= */
const STRINGS = 6;
const FRETS = 14;

const SUBDIVISIONS = {
  quarter: 1,
  eighth: 2,
  sixteenth: 4,
};

/* =========================
   TIME GRID
========================= */
function getTickMs(bpm, subdivision) {
  const factor = SUBDIVISIONS[subdivision];
  return 60000 / bpm / factor;
}

function buildTickGrid(notes, subdivision) {
  const factor = SUBDIVISIONS[subdivision];

  return notes.map((n, i) => ({
    ...n,
    tick: i * factor,
  }));
}

/* =========================
   TRANSPORT ENGINE
========================= */
function useTransportEngine(bpm, subdivision) {
  const intervalRef = useRef(null);
  const [index, setIndex] = useState(0);
  const [playing, setPlaying] = useState(false);

  const start = (grid) => {
    if (!grid.length) return;

    setPlaying(true);
    setIndex(0);

    const tickMs = getTickMs(bpm, subdivision);

    intervalRef.current = setInterval(() => {
      setIndex((prev) => {
        if (prev >= grid.length - 1) {
          clearInterval(intervalRef.current);
          setPlaying(false);
          return 0;
        }
        return prev + 1;
      });
    }, tickMs);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
    setPlaying(false);
    setIndex(0);
  };

  return { start, stop, index, playing };
}

/* =========================
   APP
========================= */
export default function App() {
  const [mode, setMode] = useState("fretboard");

  const [bpm, setBpm] = useState(100);
  const [subdivision, setSubdivision] = useState("sixteenth");

  const [notes, setNotes] = useState([]);
  const [alphaApi, setAlphaApi] = useState(null);

  const playerRef = useRef(null);

  const { start, stop, index } = useTransportEngine(bpm, subdivision);

  const grid = buildTickGrid(notes, subdivision);

  /* =========================
     FREETBOARD LOGIC
  ========================= */
  const toggleNote = (string, fret) => {
    const exists = notes.find(
      (n) => n.string === string && n.fret === fret
    );

    if (exists) {
      setNotes(notes.filter((n) => n !== exists));
    } else {
      setNotes([
        ...notes,
        { string, fret, label: `N${notes.length + 1}` },
      ]);
    }
  };

  /* =========================
     ALPHATAB IMPORT
  ========================= */
  const handleFile = async (e) => {
    const file = e.target.files[0];
    const buffer = await file.arrayBuffer();

    const api = new AlphaTabApi(
      document.getElementById("alphaTab"),
      { file: buffer }
    );

    setAlphaApi(api);
    playerRef.current = api;
  };

  const playAlpha = () => alphaApi?.play();
  const stopAlpha = () => alphaApi?.stop();

  /* =========================
     RENDER
  ========================= */
  return (
    <div style={styles.wrapper}>
      {/* TOOLBAR */}
      <div style={styles.toolbar}>
        <button onClick={() => setMode("fretboard")}>
          Fretboard
        </button>
        <button onClick={() => setMode("import")}>
          Import Tab
        </button>

        <div style={{ marginLeft: 10 }}>
          BPM:
          <input
            type="number"
            value={bpm}
            onChange={(e) => setBpm(Number(e.target.value))}
            style={{ width: 70, marginLeft: 5 }}
          />
        </div>

        <select
          value={subdivision}
          onChange={(e) => setSubdivision(e.target.value)}
          style={{ marginLeft: 10 }}
        >
          <option value="quarter">1/4</option>
          <option value="eighth">1/8</option>
          <option value="sixteenth">1/16</option>
        </select>

        <button onClick={() => start(grid)}>Play Grid</button>
        <button onClick={stop}>Stop</button>
      </div>

      {/* WORKSPACE */}
      <div style={styles.workspace}>
        {mode === "fretboard" && (
          <Fretboard
            notes={notes}
            activeIndex={index}
            onToggle={toggleNote}
          />
        )}

        {mode === "import" && (
          <div>
            <input type="file" onChange={handleFile} />

            <div id="alphaTab" style={{ marginTop: 10 }} />

            <div style={{ marginTop: 10 }}>
              <button onClick={playAlpha}>Play</button>
              <button onClick={stopAlpha}>Stop</button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

/* =========================
   FREETBOARD COMPONENT
========================= */
function Fretboard({ notes, activeIndex, onToggle }) {
  return (
    <div style={styles.fretboard}>
      {Array.from({ length: STRINGS }).map((_, string) => (
        <div key={string} style={styles.stringRow}>
          {Array.from({ length: FRETS }).map((_, fret) => {
            const note = notes.find(
              (n) => n.string === string && n.fret === fret
            );

            const isActive =
              notes[activeIndex]?.string === string &&
              notes[activeIndex]?.fret === fret;

            return (
              <div
                key={fret}
                onClick={() => onToggle(string, fret)}
                style={{
                  ...styles.fret,
                  background: isActive ? "#ffe08a" : "#fff",
                }}
              >
                {note && (
                  <div style={styles.dot}>{note.label}</div>
                )}
              </div>
            );
          })}
        </div>
      ))}
    </div>
  );
}

/* =========================
   STYLES
========================= */
const styles = {
  wrapper: {
    background: "#f5f5f0",
    height: "100vh",
    display: "flex",
    flexDirection: "column",
  },

  toolbar: {
    display: "flex",
    padding: 10,
    gap: 10,
    alignItems: "center",
    flexWrap: "wrap",
  },

  workspace: {
    flex: 1,
    resize: "both",
    overflow: "auto",
    border: "2px solid #ccc",
    margin: 10,
    background: "#f5f5f0",
  },

  fretboard: {
    display: "flex",
    flexDirection: "column",
    padding: 20,
  },

  stringRow: {
    display: "flex",
  },

  fret: {
    width: 40,
    height: 30,
    border: "1px solid #999",
    position: "relative",
    cursor: "pointer",
  },

  dot: {
    position: "absolute",
    top: 5,
    left: 5,
    fontSize: 10,
    background: "#333",
    color: "#fff",
    borderRadius: 4,
    padding: "2px 4px",
  },
};
