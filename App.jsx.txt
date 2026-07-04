import React, { useState, useRef, useEffect } from "react";
import {
  Wheat,
  Sprout,
  FlaskConical,
  MapPin,
  Clock,
  FileText,
  Paperclip,
  Send,
  CheckCircle2,
  Navigation,
  Truck,
  User,
  Hash,
  ArrowRight,
  AlertCircle,
  X,
  Lock,
  LogOut,
  LayoutDashboard,
  ClipboardList,
  Timer,
} from "lucide-react";

const SUPABASE_URL = "https://ecyyekbhcsfrzawutzzs.supabase.co";
const SUPABASE_ANON_KEY =
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImVjeXlla2JoY3Nmcnphd3V0enpzIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODMxMDE5MjAsImV4cCI6MjA5ODY3NzkyMH0.P_ULSmtmjiSaN3NgqMPDaGqoeRVXyvokuA-fgaoYvYs";

async function supabaseFetch(path, options = {}) {
  const res = await fetch(`${SUPABASE_URL}${path}`, {
    ...options,
    headers: {
      apikey: SUPABASE_ANON_KEY,
      ...options.headers,
    },
  });
  if (!res.ok) {
    const text = await res.text();
    throw new Error(text || `Erro ${res.status}`);
  }
  return res;
}

const PAGE_BG = "#EDEAE1";
const CARD_BG = "#F7F5EF";
const INK = "#2B2620";
const INK_SOFT = "#6B6558";
const RULE = "#C9C2AF";

const PRODUCTS = [
  { id: "SOJA", label: "Soja", icon: Wheat, color: "#3F6B4F" },
  { id: "MILHO", label: "Milho", icon: Sprout, color: "#C68A1F" },
  { id: "FERTILIZANTE", label: "Fertilizante", icon: FlaskConical, color: "#4C6B80" },
];

const EMPTY_FORM = {
  produto: "",
  chegada: "",
  saida: "",
  descarga: "",
  nfe: "",
  cte: "",
  viagem: "",
  transportadora: "",
  motorista: "",
  placa: "",
  origem: "",
  destino: "",
  motivo: "",
};

function Perforation({ top }) {
  return (
    <div
      style={{
        position: "absolute",
        [top ? "top" : "bottom"]: -9,
        left: 0,
        right: 0,
        height: 18,
        backgroundImage: `radial-gradient(circle at 9px 9px, ${PAGE_BG} 8px, transparent 8.5px)`,
        backgroundSize: "18px 18px",
        backgroundRepeat: "repeat-x",
        pointerEvents: "none",
      }}
    />
  );
}

function Field({ label, icon: Icon, children, required }) {
  return (
    <label className="block mb-4">
      <span
        className="flex items-center gap-1.5 text-xs uppercase tracking-wider mb-1.5"
        style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
      >
        {Icon && <Icon size={13} strokeWidth={2.2} />}
        {label}
        {required && <span style={{ color: "#A8442B" }}>*</span>}
      </span>
      {children}
    </label>
  );
}

const inputBase =
  "w-full rounded-md px-3 py-2.5 text-[15px] outline-none transition-shadow";
const inputStyle = {
  background: "#FFFFFF",
  border: `1.5px solid ${RULE}`,
  color: INK,
  fontFamily: "'IBM Plex Mono', monospace",
};

export default function PatioApp() {
  const [form, setForm] = useState(EMPTY_FORM);
  const [file, setFile] = useState(null);
  const [gps, setGps] = useState(null);
  const [gpsStatus, setGpsStatus] = useState("idle"); // idle | loading | ok | error
  const [lastTicket, setLastTicket] = useState(null);
  const [history, setHistory] = useState([]);
  const [errors, setErrors] = useState([]);
  const fileInputRef = useRef(null);

  const [view, setView] = useState("motorista"); // motorista | gestor
  const [authenticated, setAuthenticated] = useState(false);
  const [accessToken, setAccessToken] = useState(null);
  const [emailInput, setEmailInput] = useState("");
  const [passwordInput, setPasswordInput] = useState("");
  const [loginError, setLoginError] = useState("");
  const [loggingIn, setLoggingIn] = useState(false);
  const [submitting, setSubmitting] = useState(false);
  const [submitError, setSubmitError] = useState("");
  const [remoteRecords, setRemoteRecords] = useState([]);
  const [loadingRecords, setLoadingRecords] = useState(false);

  const handleLogin = async () => {
    if (!emailInput || !passwordInput) {
      setLoginError("Preencha e-mail e senha.");
      return;
    }
    setLoggingIn(true);
    setLoginError("");
    try {
      const res = await supabaseFetch("/auth/v1/token?grant_type=password", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email: emailInput, password: passwordInput }),
      });
      const data = await res.json();
      setAccessToken(data.access_token);
      setAuthenticated(true);
      setPasswordInput("");
    } catch (err) {
      setLoginError("E-mail ou senha incorretos.");
    } finally {
      setLoggingIn(false);
    }
  };

  const fetchRecords = async (token) => {
    setLoadingRecords(true);
    try {
      const res = await supabaseFetch(
        "/rest/v1/registros_patio?select=*&order=enviado_em.desc",
        { headers: { Authorization: `Bearer ${token}` } }
      );
      const data = await res.json();
      setRemoteRecords(data);
    } catch (err) {
      setRemoteRecords([]);
    } finally {
      setLoadingRecords(false);
    }
  };

  const update = (key) => (e) =>
    setForm((f) => ({ ...f, [key]: e.target.value }));

  const captureGps = () => {
    setGpsStatus("loading");
    if (!navigator.geolocation) {
      setGpsStatus("error");
      return;
    }
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        setGps({
          lat: pos.coords.latitude.toFixed(5),
          lng: pos.coords.longitude.toFixed(5),
        });
        setGpsStatus("ok");
      },
      () => setGpsStatus("error"),
      { timeout: 8000 }
    );
  };

  const requiredFields = [
    "produto",
    "chegada",
    "saida",
    "descarga",
    "nfe",
    "cte",
    "viagem",
    "transportadora",
    "motorista",
    "placa",
    "origem",
    "destino",
  ]; // nota: "anexo" fica de fora daqui de propósito só durante o teste no preview

  const handleSubmit = async () => {
    const missing = requiredFields.filter((k) => !form[k]);
    if (missing.length) {
      setErrors(missing);
      return;
    }
    setErrors([]);
    setSubmitError("");
    setSubmitting(true);

    try {
      if (gpsStatus !== "ok") captureGps();

      // 1. Upload do PDF pro storage (se tiver anexo — opcional só neste teste)
      let filePath = null;
      if (file) {
        filePath = `${Date.now()}_${file.name.replace(/\s+/g, "_")}`;
        await supabaseFetch(`/storage/v1/object/comprovantes/${filePath}`, {
          method: "POST",
          headers: {
            Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
            "Content-Type": file.type || "application/pdf",
          },
          body: file,
        });
      }

      // 2. Inserir o registro na tabela
      const payload = {
        produto: form.produto,
        chegada: new Date(form.chegada).toISOString(),
        saida: new Date(form.saida).toISOString(),
        descarga: new Date(form.descarga).toISOString(),
        nfe: form.nfe,
        cte: form.cte,
        viagem: form.viagem,
        transportadora: form.transportadora,
        motorista: form.motorista,
        placa: form.placa,
        origem: form.origem,
        destino: form.destino,
        motivo: form.motivo,
        arquivo_url: filePath,
        gps_lat: gps?.lat ? Number(gps.lat) : null,
        gps_lng: gps?.lng ? Number(gps.lng) : null,
      };

      const res = await supabaseFetch("/rest/v1/registros_patio", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
          "Content-Type": "application/json",
          Prefer: "return=representation",
        },
        body: JSON.stringify(payload),
      });
      const [inserted] = await res.json();

      const ticket = {
        numero: String(inserted.numero).padStart(5, "0"),
        enviadoEm: new Date(inserted.enviado_em).toLocaleString("pt-BR"),
        gps,
        arquivo: file ? file.name : "(nenhum anexo — modo teste)",
        ...form,
      };
      setLastTicket(ticket);
      setHistory((h) => [ticket, ...h].slice(0, 5));
      setForm(EMPTY_FORM);
      setFile(null);
      setGps(null);
      setGpsStatus("idle");
      if (fileInputRef.current) fileInputRef.current.value = "";
    } catch (err) {
      setSubmitError(
        "Não foi possível enviar o registro. Verifique sua conexão e tente novamente."
      );
    } finally {
      setSubmitting(false);
    }
  };

  useEffect(() => {
    if (authenticated && accessToken) fetchRecords(accessToken);
  }, [authenticated, accessToken]);

  const selectedProduct = PRODUCTS.find((p) => p.id === form.produto);

  return (
    <div
      className="min-h-screen w-full flex justify-center px-4 py-8"
      style={{ background: PAGE_BG }}
    >
      <style>{`
        input:focus, textarea:focus, select:focus { box-shadow: 0 0 0 3px rgba(63,107,79,0.15); border-color: #3F6B4F !important; }
      `}</style>

      <div className="w-full max-w-md">
        {/* Header */}
        <div className="mb-6 text-center">
          <div
            className="inline-flex items-center gap-2 px-3 py-1 rounded-full mb-2"
            style={{ background: "#DFE6DE", color: "#3F6B4F" }}
          >
            <Truck size={14} />
            <span
              className="text-[11px] uppercase tracking-widest font-semibold"
              style={{ fontFamily: "'IBM Plex Mono', monospace" }}
            >
              Protótipo · sem envio real
            </span>
          </div>
          <h1
            style={{
              fontFamily: "'Oswald', sans-serif",
              color: INK,
              letterSpacing: "0.01em",
            }}
            className="text-3xl font-semibold uppercase"
          >
            Controle de Pátio
          </h1>
          <p
            className="text-sm mt-1"
            style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
          >
            Registro de estadia · carga de grãos e fertilizante
          </p>
        </div>

        {/* Tab switcher */}
        <div
          className="flex rounded-md p-1 mb-6"
          style={{ background: "#E2DFD3", border: `1px solid ${RULE}` }}
        >
          <button
            type="button"
            onClick={() => setView("motorista")}
            className="flex-1 flex items-center justify-center gap-1.5 py-2 rounded text-sm font-medium transition-all"
            style={{
              background: view === "motorista" ? CARD_BG : "transparent",
              color: view === "motorista" ? INK : INK_SOFT,
              fontFamily: "'IBM Plex Mono', monospace",
              boxShadow: view === "motorista" ? "0 1px 2px rgba(0,0,0,0.08)" : "none",
            }}
          >
            <Truck size={14} /> Motorista
          </button>
          <button
            type="button"
            onClick={() => setView("gestor")}
            className="flex-1 flex items-center justify-center gap-1.5 py-2 rounded text-sm font-medium transition-all"
            style={{
              background: view === "gestor" ? CARD_BG : "transparent",
              color: view === "gestor" ? INK : INK_SOFT,
              fontFamily: "'IBM Plex Mono', monospace",
              boxShadow: view === "gestor" ? "0 1px 2px rgba(0,0,0,0.08)" : "none",
            }}
          >
            <LayoutDashboard size={14} /> Painel gestor
          </button>
        </div>

        {view === "gestor" && !authenticated && (
          <div className="relative mb-6">
            <Perforation top />
            <div
              className="rounded-lg px-5 py-8 flex flex-col items-center text-center"
              style={{ background: CARD_BG, border: `1px solid ${RULE}` }}
            >
              <div
                className="flex items-center justify-center w-11 h-11 rounded-full mb-3"
                style={{ background: "#DFE6DE" }}
              >
                <Lock size={18} color="#3F6B4F" />
              </div>
              <h2
                className="text-lg font-semibold mb-1"
                style={{ fontFamily: "'Oswald', sans-serif", color: INK }}
              >
                ÁREA DO GESTOR
              </h2>
              <p
                className="text-xs mb-5"
                style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
              >
                Entre com o login cadastrado para ver todos os registros
              </p>
              <input
                type="email"
                className={inputBase + " text-center mb-2"}
                style={inputStyle}
                placeholder="E-mail"
                value={emailInput}
                onChange={(e) => {
                  setEmailInput(e.target.value);
                  setLoginError("");
                }}
              />
              <input
                type="password"
                className={inputBase + " text-center mb-3"}
                style={inputStyle}
                placeholder="Senha"
                value={passwordInput}
                onChange={(e) => {
                  setPasswordInput(e.target.value);
                  setLoginError("");
                }}
                onKeyDown={(e) => e.key === "Enter" && handleLogin()}
              />
              {loginError && (
                <p
                  className="text-xs mb-3"
                  style={{ color: "#A8442B", fontFamily: "'IBM Plex Mono', monospace" }}
                >
                  {loginError}
                </p>
              )}
              <button
                type="button"
                onClick={handleLogin}
                disabled={loggingIn}
                className="w-full flex items-center justify-center gap-2 rounded-md py-3 font-semibold text-white"
                style={{ background: "#3F6B4F", fontFamily: "'Oswald', sans-serif", opacity: loggingIn ? 0.7 : 1 }}
              >
                <Lock size={15} /> {loggingIn ? "ENTRANDO..." : "ENTRAR"}
              </button>
            </div>
            <Perforation />
          </div>
        )}

        {view === "gestor" && authenticated && (
          <ManagerPanel
            records={remoteRecords}
            loading={loadingRecords}
            onLogout={() => {
              setAuthenticated(false);
              setAccessToken(null);
              setView("motorista");
            }}
          />
        )}

        {view === "motorista" && (
        <>
        {/* Ticket card */}
        <div className="relative mb-6">
          <Perforation top />
          <div
            className="rounded-lg px-5 pt-7 pb-6 shadow-sm"
            style={{ background: CARD_BG, border: `1px solid ${RULE}` }}
          >
            {/* Ticket number strip */}
            <div
              className="flex items-center justify-between pb-4 mb-5"
              style={{ borderBottom: `1px dashed ${RULE}` }}
            >
              <div className="flex items-center gap-1.5" style={{ color: INK_SOFT }}>
                <Hash size={13} />
                <span
                  className="text-xs"
                  style={{ fontFamily: "'IBM Plex Mono', monospace" }}
                >
                  novo registro de estadia
                </span>
              </div>
            </div>

            {/* Product selector */}
            <div className="mb-5">
              <span
                className="flex items-center gap-1.5 text-xs uppercase tracking-wider mb-2"
                style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
              >
                Tipo de produto <span style={{ color: "#A8442B" }}>*</span>
              </span>
              <div className="grid grid-cols-3 gap-2">
                {PRODUCTS.map((p) => {
                  const active = form.produto === p.id;
                  const Icon = p.icon;
                  return (
                    <button
                      key={p.id}
                      type="button"
                      onClick={() => setForm((f) => ({ ...f, produto: p.id }))}
                      className="flex flex-col items-center gap-1.5 py-3 rounded-md transition-all"
                      style={{
                        background: active ? p.color : "#FFFFFF",
                        border: `1.5px solid ${active ? p.color : RULE}`,
                      }}
                    >
                      <Icon
                        size={18}
                        color={active ? "#FFFFFF" : p.color}
                        strokeWidth={2.2}
                      />
                      <span
                        className="text-[11px] font-medium"
                        style={{
                          color: active ? "#FFFFFF" : INK,
                          fontFamily: "'IBM Plex Mono', monospace",
                        }}
                      >
                        {p.label}
                      </span>
                    </button>
                  );
                })}
              </div>
            </div>

            {/* Datetime fields */}
            <Field label="Chegada" icon={Clock} required>
              <input
                type="datetime-local"
                className={inputBase}
                style={inputStyle}
                value={form.chegada}
                onChange={update("chegada")}
              />
            </Field>
            <Field label="Saída" icon={Clock} required>
              <input
                type="datetime-local"
                className={inputBase}
                style={inputStyle}
                value={form.saida}
                onChange={update("saida")}
              />
            </Field>
            <Field label="Data e hora da descarga" icon={Clock} required>
              <input
                type="datetime-local"
                className={inputBase}
                style={inputStyle}
                value={form.descarga}
                onChange={update("descarga")}
              />
            </Field>

            <div style={{ borderTop: `1px dashed ${RULE}` }} className="pt-4 mt-1" />

            <Field label="Número da NFe" icon={FileText} required>
              <input
                type="text"
                inputMode="numeric"
                className={inputBase}
                style={inputStyle}
                placeholder="000000000"
                value={form.nfe}
                onChange={update("nfe")}
              />
            </Field>
            <Field label="Número do CTe" icon={FileText} required>
              <input
                type="text"
                inputMode="numeric"
                className={inputBase}
                style={inputStyle}
                placeholder="000000000"
                value={form.cte}
                onChange={update("cte")}
              />
            </Field>
            <Field label="Nº da viagem contratada" icon={Hash} required>
              <input
                type="text"
                className={inputBase}
                style={inputStyle}
                value={form.viagem}
                onChange={update("viagem")}
              />
            </Field>

            <div style={{ borderTop: `1px dashed ${RULE}` }} className="pt-4 mt-1" />

            <Field label="Transportadora" icon={Truck} required>
              <input
                type="text"
                className={inputBase}
                style={inputStyle}
                value={form.transportadora}
                onChange={update("transportadora")}
              />
            </Field>
            <Field label="Motorista" icon={User} required>
              <input
                type="text"
                className={inputBase}
                style={inputStyle}
                value={form.motorista}
                onChange={update("motorista")}
              />
            </Field>
            <Field label="Placa" icon={Truck} required>
              <input
                type="text"
                className={inputBase}
                style={{ ...inputStyle, textTransform: "uppercase" }}
                placeholder="ABC1D23"
                value={form.placa}
                onChange={update("placa")}
              />
            </Field>

            <div style={{ borderTop: `1px dashed ${RULE}` }} className="pt-4 mt-1" />

            <div className="grid grid-cols-2 gap-3">
              <Field label="Origem" icon={MapPin} required>
                <input
                  type="text"
                  className={inputBase}
                  style={inputStyle}
                  value={form.origem}
                  onChange={update("origem")}
                />
              </Field>
              <Field label="Destino" icon={ArrowRight} required>
                <input
                  type="text"
                  className={inputBase}
                  style={inputStyle}
                  value={form.destino}
                  onChange={update("destino")}
                />
              </Field>
            </div>

            <Field label="Motivo de ter ficado parado">
              <textarea
                rows={2}
                className={inputBase}
                style={{ ...inputStyle, resize: "none" }}
                placeholder="Ex: fila de descarga, chuva, falta de nota..."
                value={form.motivo}
                onChange={update("motivo")}
              />
            </Field>

            <div style={{ borderTop: `1px dashed ${RULE}` }} className="pt-4 mt-1" />

            {/* File attach */}
            <label className="block mb-4">
              <span
                className="flex items-center gap-1.5 text-xs uppercase tracking-wider mb-1.5"
                style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
              >
                <Paperclip size={13} />
                Anexo PDF <span style={{ color: INK_SOFT, textTransform: "none", letterSpacing: 0 }}>(opcional neste teste)</span>
              </span>
              <div
                className="flex items-center justify-between rounded-md px-3 py-2.5"
                style={inputStyle}
              >
                <span
                  className="text-sm truncate"
                  style={{ color: file ? INK : INK_SOFT }}
                >
                  {file ? file.name : "Toque para anexar comprovante..."}
                </span>
                {file && (
                  <button
                    type="button"
                    onClick={(e) => {
                      e.preventDefault();
                      e.stopPropagation();
                      setFile(null);
                    }}
                  >
                    <X size={15} color={INK_SOFT} />
                  </button>
                )}
              </div>
              <input
                ref={fileInputRef}
                type="file"
                accept="application/pdf"
                style={{
                  position: "absolute",
                  width: 1,
                  height: 1,
                  padding: 0,
                  margin: -1,
                  overflow: "hidden",
                  clip: "rect(0,0,0,0)",
                  border: 0,
                }}
                onChange={(e) => setFile(e.target.files?.[0] || null)}
              />
            </label>

            {/* GPS capture */}
            <button
              type="button"
              onClick={captureGps}
              className="w-full flex items-center justify-center gap-2 rounded-md py-2.5 mb-2 text-sm font-medium"
              style={{
                background: gpsStatus === "ok" ? "#DFE6DE" : "#FFFFFF",
                border: `1.5px solid ${gpsStatus === "ok" ? "#3F6B4F" : RULE}`,
                color: gpsStatus === "ok" ? "#3F6B4F" : INK,
                fontFamily: "'IBM Plex Mono', monospace",
              }}
            >
              <Navigation size={15} />
              {gpsStatus === "idle" && "Capturar localização atual"}
              {gpsStatus === "loading" && "Obtendo coordenadas..."}
              {gpsStatus === "ok" && `Local: ${gps.lat}, ${gps.lng}`}
              {gpsStatus === "error" && "Localização indisponível — tentar de novo"}
            </button>
            <p
              className="text-[11px] mb-4"
              style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
            >
              Data/hora do envio e coordenadas são capturadas automaticamente ao enviar.
            </p>

            {errors.length > 0 && (
              <div
                className="flex items-start gap-2 rounded-md px-3 py-2.5 mb-4 text-sm"
                style={{ background: "#F3E1DA", color: "#A8442B" }}
              >
                <AlertCircle size={16} className="shrink-0 mt-0.5" />
                <span>Preencha todos os campos obrigatórios (*) e anexe o PDF antes de enviar.</span>
              </div>
            )}

            {submitError && (
              <div
                className="flex items-start gap-2 rounded-md px-3 py-2.5 mb-4 text-sm"
                style={{ background: "#F3E1DA", color: "#A8442B" }}
              >
                <AlertCircle size={16} className="shrink-0 mt-0.5" />
                <span>{submitError}</span>
              </div>
            )}

            <button
              type="button"
              onClick={handleSubmit}
              disabled={submitting}
              className="w-full flex items-center justify-center gap-2 rounded-md py-3 font-semibold text-white"
              style={{
                background: selectedProduct ? selectedProduct.color : "#3F6B4F",
                fontFamily: "'Oswald', sans-serif",
                letterSpacing: "0.02em",
                opacity: submitting ? 0.7 : 1,
              }}
            >
              <Send size={16} />
              {submitting ? "ENVIANDO..." : "ENVIAR REGISTRO"}
            </button>
          </div>
          <Perforation />
        </div>

        {/* Confirmation */}
        {lastTicket && (
          <div className="relative mb-6">
            <Perforation top />
            <div
              className="rounded-lg px-5 py-6"
              style={{ background: "#DFE6DE", border: "1px solid #3F6B4F" }}
            >
              <div className="flex items-center gap-2 mb-3">
                <CheckCircle2 size={18} color="#3F6B4F" />
                <span
                  className="font-semibold text-sm"
                  style={{ color: "#3F6B4F", fontFamily: "'Oswald', sans-serif" }}
                >
                  REGISTRO Nº {lastTicket.numero} ENVIADO
                </span>
              </div>
              <ul
                className="text-[13px] space-y-1"
                style={{ color: "#2B4030", fontFamily: "'IBM Plex Mono', monospace" }}
              >
                <li>→ E-mail enviado ao gestor com anexo "{lastTicket.arquivo}"</li>
                <li>→ Linha adicionada a controle_patio.xlsx</li>
                <li>→ Enviado em {lastTicket.enviadoEm}</li>
                <li>
                  → GPS:{" "}
                  {lastTicket.gps
                    ? `${lastTicket.gps.lat}, ${lastTicket.gps.lng}`
                    : "não capturado neste ambiente"}
                </li>
              </ul>
            </div>
            <Perforation />
          </div>
        )}

        {/* History */}
        {history.length > 0 && (
          <div>
            <h2
              className="text-xs uppercase tracking-widest mb-2 px-1"
              style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
            >
              Últimos registros
            </h2>
            <div className="space-y-2">
              {history.map((h) => {
                const prod = PRODUCTS.find((p) => p.id === h.produto);
                const Icon = prod?.icon || Wheat;
                return (
                  <div
                    key={h.numero}
                    className="flex items-center gap-3 rounded-md px-3 py-2.5"
                    style={{ background: CARD_BG, border: `1px solid ${RULE}` }}
                  >
                    <div
                      className="flex items-center justify-center w-8 h-8 rounded-full shrink-0"
                      style={{ background: prod?.color || "#3F6B4F" }}
                    >
                      <Icon size={14} color="#fff" />
                    </div>
                    <div className="min-w-0 flex-1">
                      <p
                        className="text-sm font-medium truncate"
                        style={{ color: INK }}
                      >
                        {h.transportadora} · {h.placa}
                      </p>
                      <p
                        className="text-[11px] truncate"
                        style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
                      >
                        {h.origem} → {h.destino} · Nº{h.numero}
                      </p>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>
        )}
        </>
        )}
      </div>
    </div>
  );
}

function computeStopMinutes(chegada, saida) {
  if (!chegada || !saida) return null;
  const diff = (new Date(saida) - new Date(chegada)) / 60000;
  if (isNaN(diff) || diff < 0) return null;
  return Math.round(diff);
}

function formatMinutes(mins) {
  if (mins == null) return "—";
  const h = Math.floor(mins / 60);
  const m = mins % 60;
  return h > 0 ? `${h}h${String(m).padStart(2, "0")}` : `${m}min`;
}

function ManagerPanel({ records, loading, onLogout }) {
  const totalPorProduto = PRODUCTS.map((p) => ({
    ...p,
    count: records.filter((h) => h.produto === p.id).length,
  }));

  return (
    <div className="relative mb-6">
      <Perforation top />
      <div
        className="rounded-lg px-5 pt-6 pb-6"
        style={{ background: CARD_BG, border: `1px solid ${RULE}` }}
      >
        <div className="flex items-center justify-between mb-5">
          <div className="flex items-center gap-2">
            <ClipboardList size={17} color={INK} />
            <h2
              className="text-lg font-semibold"
              style={{ fontFamily: "'Oswald', sans-serif", color: INK }}
            >
              PAINEL GESTOR
            </h2>
          </div>
          <button
            type="button"
            onClick={onLogout}
            className="flex items-center gap-1 text-xs"
            style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
          >
            <LogOut size={13} /> Sair
          </button>
        </div>

        {/* Summary cards */}
        <div className="grid grid-cols-4 gap-2 mb-5">
          <div
            className="rounded-md py-3 px-2 text-center"
            style={{ background: "#FFFFFF", border: `1px solid ${RULE}` }}
          >
            <p className="text-lg font-semibold" style={{ color: INK, fontFamily: "'Oswald', sans-serif" }}>
              {records.length}
            </p>
            <p className="text-[10px] uppercase" style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}>
              Total
            </p>
          </div>
          {totalPorProduto.map((p) => (
            <div
              key={p.id}
              className="rounded-md py-3 px-2 text-center"
              style={{ background: "#FFFFFF", border: `1px solid ${p.color}55` }}
            >
              <p className="text-lg font-semibold" style={{ color: p.color, fontFamily: "'Oswald', sans-serif" }}>
                {p.count}
              </p>
              <p className="text-[10px] uppercase" style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}>
                {p.label}
              </p>
            </div>
          ))}
        </div>

        {loading ? (
          <div
            className="rounded-md py-8 text-center text-sm"
            style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
          >
            Carregando registros...
          </div>
        ) : records.length === 0 ? (
          <div
            className="rounded-md py-8 text-center text-sm"
            style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
          >
            Nenhum registro enviado ainda. Registros feitos na aba "Motorista" vão aparecer aqui.
          </div>
        ) : (
          <div className="space-y-2">
            {records.map((h) => {
              const prod = PRODUCTS.find((p) => p.id === h.produto);
              const Icon = prod?.icon || Wheat;
              const stopMinutes = computeStopMinutes(h.chegada, h.saida);
              return (
                <div
                  key={h.id || h.numero}
                  className="rounded-md px-3 py-3"
                  style={{ background: "#FFFFFF", border: `1px solid ${RULE}` }}
                >
                  <div className="flex items-center justify-between mb-2">
                    <div className="flex items-center gap-2">
                      <div
                        className="flex items-center justify-center w-6 h-6 rounded-full shrink-0"
                        style={{ background: prod?.color || "#3F6B4F" }}
                      >
                        <Icon size={12} color="#fff" />
                      </div>
                      <span
                        className="text-sm font-medium"
                        style={{ color: INK, fontFamily: "'IBM Plex Mono', monospace" }}
                      >
                        Nº {String(h.numero).padStart(5, "0")}
                      </span>
                    </div>
                    <div className="flex items-center gap-1" style={{ color: INK_SOFT }}>
                      <Timer size={12} />
                      <span className="text-[11px]" style={{ fontFamily: "'IBM Plex Mono', monospace" }}>
                        {formatMinutes(stopMinutes)}
                      </span>
                    </div>
                  </div>
                  <p className="text-sm" style={{ color: INK }}>
                    {h.transportadora} · {h.motorista} · {h.placa}
                  </p>
                  <p
                    className="text-[11px] mt-0.5"
                    style={{ color: INK_SOFT, fontFamily: "'IBM Plex Mono', monospace" }}
                  >
                    {h.origem} → {h.destino} · NFe {h.nfe} · CTe {h.cte}
                  </p>
                  {h.motivo && (
                    <p
                      className="text-[11px] mt-1 italic"
                      style={{ color: INK_SOFT }}
                    >
                      Motivo: {h.motivo}
                    </p>
                  )}
                  <p
                    className="text-[10px] mt-1.5"
                    style={{ color: RULE, fontFamily: "'IBM Plex Mono', monospace" }}
                  >
                    Anexo: {h.arquivo_url} · Enviado em{" "}
                    {h.enviado_em ? new Date(h.enviado_em).toLocaleString("pt-BR") : "—"}
                  </p>
                </div>
              );
            })}
          </div>
        )}
      </div>
      <Perforation />
    </div>
  );
}
