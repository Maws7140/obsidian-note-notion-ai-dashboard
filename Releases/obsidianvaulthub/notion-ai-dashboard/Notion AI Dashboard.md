---
cssclasses:
  - hide-toolbar
  - hide-properties
mode: read_only
modified: 2026-06-12
---
```datacorejsx
// -- Configuration ----------------------------------------------------------
// Path where the Agent Client plugin stores its data file.
// Default for the Agent Client plugin: .obsidian/plugins/agent-client/data.json
const AGENT_DATA_PATH = ".obsidian/plugins/agent-client/data.json";

// Folder where agent prompt notes live. When you click an agent avatar,
// the corresponding prompt note is opened (or created) here.
const PROMPTS_FOLDER = "Meta";

// Where new chat notes are created when you press "New chat".
const CHAT_NOTE_PATH = "AI Chats/New Chat.md";
// ---------------------------------------------------------------------------

return function NotionAIAssistant() {
    const [agents, setAgents] = dc.useState([]);
    const [sessions, setSessions] = dc.useState([]);
    const [loading, setLoading] = dc.useState(true);
    const [showMenu, setShowMenu] = dc.useState(false);

    dc.useEffect(() => {
        let isMounted = true;

        async function fetchData() {
            try {
                const dataRaw = await app.vault.adapter.read(AGENT_DATA_PATH);
                const data = JSON.parse(dataRaw);

                if (isMounted) {
                    const allAgents = [...(data.customAgents || [])];

                    // Pull top-level agents / built-ins from root-level keys
                    Object.keys(data).forEach(key => {
                        const val = data[key];
                        if (val && typeof val === "object" && val.id && val.displayName) {
                            if (!allAgents.find(a => a.id === val.id)) {
                                allAgents.push(val);
                            }
                        }
                    });

                    setAgents(allAgents);
                    setSessions(data.savedSessions || []);
                    setLoading(false);
                }
            } catch (e) {
                console.error("Failed to load agent data", e);
                if (isMounted) setLoading(false);
            }
        }

        fetchData();
        return () => { isMounted = false; };
    }, []);

    // Opens (or creates) the prompt note for an agent.
    const openPromptNote = async (agent) => {
        let promptPath = `${PROMPTS_FOLDER}/${agent.displayName} Agent Prompt.md`;

        // Try finding prompt path in agent args
        const arg = agent.args && agent.args.find(
            a => typeof a === "string" && a.includes("model_instructions_file=")
        );
        if (arg) {
            const fullPath = arg.split("model_instructions_file=")[1];
            promptPath = fullPath
                .replace(app.vault.adapter.getBasePath() + "/", "")
                .replace(/\\/g, "/")
                .replace(/^[\\\/]+/, "");
        }

        // Try finding prompt path in agent env
        if (agent.env) {
            const envKeys = ["GEMINI_SYSTEM_MD", "OPENCODE_SYSTEM_PROMPT_FILE", "SYSTEM_PROMPT_FILE"];
            const envVar = agent.env.find(e => envKeys.includes(e.key));
            if (envVar) {
                promptPath = envVar.value
                    .replace(app.vault.adapter.getBasePath() + "/", "")
                    .replace(/\\/g, "/")
                    .replace(/^[\\\/]+/, "");
            }
        }

        let file = app.vault.getAbstractFileByPath(promptPath);
        if (!file) {
            file = await app.vault.create(
                promptPath,
                `---\ntitle: ${agent.displayName} Agent Prompt\ntags: [agent, prompt]\n---\n`
            );
        }
        app.workspace.getLeaf(false).openFile(file);
    };

    // Creates a stub prompt note for a brand-new agent
    const createNewAgent = async () => {
        const name = "New Agent " + Date.now();
        const promptPath = `${PROMPTS_FOLDER}/${name} Agent Prompt.md`;
        const file = await app.vault.create(
            promptPath,
            `---\ntitle: ${name} Agent Prompt\ntags: [agent, prompt]\n---\n`
        );
        app.workspace.getLeaf(false).openFile(file);
        new Notice("Note created. Configure this agent in Agent Client settings.");
    };

    // Opens or creates the chat note
    const openChat = async () => {
        let file = app.vault.getAbstractFileByPath(CHAT_NOTE_PATH);
        if (!file) {
            // Ensure parent folder exists
            const parts = CHAT_NOTE_PATH.split("/");
            parts.pop();
            const folder = parts.join("/");
            if (folder && !(await app.vault.adapter.exists(folder))) {
                await app.vault.createFolder(folder);
            }
            file = await app.vault.create(
                CHAT_NOTE_PATH,
                `---\ntitle: New Chat\ntags: [ai-chat]\n---\n`
            );
        }
        app.workspace.getLeaf(false).openFile(file);
    };

    // Group sessions by recency
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const thirtyDaysAgo = new Date(today.getTime() - 30 * 24 * 60 * 60 * 1000);

    const groups = { Today: [], "Past 30 days": [], Older: [] };
    sessions.forEach(s => {
        const d = new Date(s.updatedAt);
        if (d >= today) groups["Today"].push(s);
        else if (d >= thirtyDaysAgo) groups["Past 30 days"].push(s);
        else groups["Older"].push(s);
    });

    // Returns avatar info for a session/agent, falling back gracefully
    const getAgentAvatar = (agentId) => {
        const agent = agents.find(a => a.id === agentId);
        if (agent && agent.avatarImage) {
            const path = agent.avatarImage.replace(/\\/g, "/").replace(/^[\\\/]+/, "");
            const file = app.vault.getAbstractFileByPath(path);
            if (file) return { type: "url", value: app.vault.getResourcePath(file) };
        }
        // Well-known provider icons
        if (agentId?.includes("gemini"))
            return { type: "url", value: "https://www.gstatic.com/lamda/images/favicon_v1_150160d13988652cbf5e.svg" };
        if (agentId?.includes("claude"))
            return { type: "url", value: "https://claude.ai/images/claude_favicon.png" };
        if (agentId?.includes("codex"))   return { type: "emoji", value: "??" };
        if (agentId?.includes("image"))   return { type: "emoji", value: "???" };
        if (agentId?.includes("opencode")) return { type: "emoji", value: "??" };
        return null;
    };

    const formatTime = (dateStr) => {
        const d = new Date(dateStr);
        const diffMs = Date.now() - d;
        const diffMins = Math.floor(diffMs / 60000);
        if (diffMins < 60) return diffMins <= 1 ? "Just now" : `${diffMins}m`;
        const diffHours = Math.floor(diffMins / 60);
        if (diffHours < 24) return `${diffHours}h`;
        const diffDays = Math.floor(diffHours / 24);
        if (diffDays < 30) return `${diffDays}d`;
        const diffWeeks = Math.floor(diffDays / 7);
        if (diffWeeks < 4) return `${diffWeeks}w`;
        return d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
    };

    if (loading) {
        return (
            <div style={{ color: "var(--text-muted)", textAlign: "center" }}>
                Loading Obsidian AI...
            </div>
        );
    }

    return (
        <div
            className="audio-mgr"
            style={{ minHeight: "calc(100vh - 80px)", display: "flex", flexDirection: "column", width: "100%", boxSizing: "border-box" }}
        >
            <div className="am-body" style={{ flex: 1, paddingBottom: "16px" }}>

                {/* -- Agent row -- */}
                <div className="am-bucket">
                    <div className="am-bucket-label" style={{ paddingLeft: "2px", paddingRight: "2px" }}>
                        Obsidian AI Assistant
                    </div>

                    <div style={{ display: "flex", gap: "12px", overflowX: "auto", padding: "4px 2px 12px" }}>
                        {agents.map(agent => {
                            const avatar = getAgentAvatar(agent.id);
                            return (
                                <div
                                    key={agent.id}
                                    onClick={() => openPromptNote(agent)}
                                    style={{ cursor: "pointer", display: "flex", flexDirection: "column", alignItems: "center", minWidth: "48px" }}
                                >
                                    <div style={{
                                        width: "36px", height: "36px", borderRadius: "50%",
                                        backgroundColor: "var(--background-secondary)",
                                        display: "flex", alignItems: "center", justifyContent: "center",
                                        marginBottom: "6px", overflow: "hidden",
                                        border: "1px solid var(--background-modifier-border)"
                                    }}>
                                        {avatar?.type === "url" ? (
                                            <img src={avatar.value} style={{ width: "100%", height: "100%", objectFit: "cover" }} />
                                        ) : (
                                            <span style={{ fontSize: "18px" }}>{avatar?.value || "??"}</span>
                                        )}
                                    </div>
                                    <span style={{
                                        fontSize: "0.75rem", textAlign: "center", color: "var(--text-muted)",
                                        overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap", width: "100%"
                                    }}>
                                        {agent.displayName || agent.id}
                                    </span>
                                </div>
                            );
                        })}

                        {/* New agent button */}
                        <div onClick={createNewAgent} style={{ cursor: "pointer", display: "flex", flexDirection: "column", alignItems: "center", minWidth: "48px" }}>
                            <div style={{
                                width: "36px", height: "36px", borderRadius: "50%",
                                backgroundColor: "var(--background-secondary)",
                                border: "1px dashed var(--background-modifier-border)",
                                display: "flex", alignItems: "center", justifyContent: "center", marginBottom: "6px"
                            }}>
                                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="var(--text-muted)" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                                    <path d="M12 5v14M5 12h14" />
                                </svg>
                            </div>
                            <span style={{ fontSize: "0.75rem", textAlign: "center", color: "var(--text-muted)" }}>New</span>
                        </div>
                    </div>
                </div>

                {/* -- Session groups -- */}
                {Object.entries(groups).map(([groupName, groupSessions]) => {
                    if (groupSessions.length === 0) return null;
                    return (
                        <div key={groupName} className="am-bucket">
                            <div className="am-bucket-label" style={{ textTransform: "uppercase" }}>{groupName}</div>
                            <div className="am-bucket-items">
                                {groupSessions
                                    .sort((a, b) => new Date(b.updatedAt) - new Date(a.updatedAt))
                                    .map(s => {
                                        const avatar = getAgentAvatar(s.agentId);
                                        return (
                                            <div key={s.sessionId} className="am-item" onClick={openChat}>
                                                <div className="am-item-icon">
                                                    {avatar?.type === "url" ? (
                                                        <img src={avatar.value} style={{ width: "14px", height: "14px", borderRadius: "3px", objectFit: "cover" }} />
                                                    ) : (
                                                        <span>{avatar?.value || (
                                                            <svg viewBox="0 0 24 24"><path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z" /></svg>
                                                        )}</span>
                                                    )}
                                                </div>
                                                <span className="am-item-title">{s.title || "New Chat"}</span>
                                                <span className="am-item-time">{formatTime(s.updatedAt)}</span>
                                            </div>
                                        );
                                    })}
                            </div>
                        </div>
                    );
                })}

            </div>

            {/* -- Sticky bottom bar -- */}
            <div style={{
                position: "sticky", bottom: 0, zIndex: 10, marginTop: "auto",
                background: "inherit", display: "flex", gap: "8px",
                alignItems: "center", paddingTop: "12px", paddingBottom: "12px",
            }}>
                <button onClick={openChat} style={{
                    flex: 1, height: "32px", backgroundColor: "var(--background-secondary)",
                    border: "1px solid var(--background-modifier-border)", borderRadius: "16px",
                    padding: "0 16px", color: "var(--text-normal)", display: "flex",
                    alignItems: "center", justifyContent: "center", gap: "8px",
                    cursor: "pointer", fontSize: "0.85rem", fontWeight: 500
                }}>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                        <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z" />
                    </svg>
                    New chat
                </button>

                <div style={{ position: "relative" }}>
                    <button onClick={() => setShowMenu(!showMenu)} style={{
                        width: "32px", height: "32px", borderRadius: "50%",
                        backgroundColor: "var(--background-secondary)",
                        border: "1px solid var(--background-modifier-border)",
                        color: "var(--text-normal)", display: "flex", alignItems: "center",
                        justifyContent: "center", cursor: "pointer"
                    }}>
                        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
                            <path d="M12 20h9" />
                            <path d="M16.5 3.5a2.121 2.121 0 0 1 3 3L7 19l-4 1 1-4L16.5 3.5z" />
                        </svg>
                    </button>

                    {showMenu && (
                        <div style={{
                            position: "absolute", bottom: "40px", right: "0",
                            backgroundColor: "var(--background-secondary)",
                            border: "1px solid var(--background-modifier-border)",
                            borderRadius: "8px", padding: "4px", display: "flex",
                            flexDirection: "column", minWidth: "140px",
                            boxShadow: "var(--shadow-s)", zIndex: 100
                        }}>
                            <div className="am-item" onClick={() => { app.commands.executeCommandById("file-explorer:new"); setShowMenu(false); }} style={{ gridTemplateColumns: "1fr" }}>
                                <span className="am-item-title">New note</span>
                            </div>
                            <div className="am-item" onClick={() => { app.commands.executeCommandById("audio-recorder:start"); setShowMenu(false); }} style={{ gridTemplateColumns: "1fr" }}>
                                <span className="am-item-title">Recording</span>
                            </div>
                            <div className="am-item" onClick={() => { app.commands.executeCommandById("datacore:create-base"); setShowMenu(false); }} style={{ gridTemplateColumns: "1fr" }}>
                                <span className="am-item-title">Base</span>
                            </div>
                        </div>
                    )}
                </div>
            </div>
        </div>
    );
}
```
