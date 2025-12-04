import React, { useState, useEffect, useRef } from "react";
import {
  Users,
  Send,
  Clock,
  AlertCircle,
  CheckCircle,
  RefreshCw,
  Search,
  ChevronLeft,
  ChevronRight,
  Trash2,
  Mail,
  Filter,
  Download,
  Repeat,
  Upload,
} from "lucide-react";

const API_URL_ADMIN_SEND_INVITE_LINK = "https://bandymoot.com/hive/v2/api/admin/invite";
const API_URL_ADMIN_GET_INVITES = "https://bandymoot.com/hive/v2/api/admin/invites";

const AdminCandidateManagement = () => {
  const [adminCandidateEmail, setAdminCandidateEmail] = useState("");
  const [adminCandidateName, setAdminCandidateName] = useState("");
  const [adminSelectedPosition, setAdminSelectedPosition] = useState("Aptitude Round");
  const [inviteHistory, setInviteHistory] = useState([]);
  const [filteredHistory, setFilteredHistory] = useState([]);
  const [showResendConfirm, setShowResendConfirm] = useState(false);
  const [pendingInvite, setPendingInvite] = useState(null);
  const [loading, setLoading] = useState(false);
  const [isLoadingHistory, setIsLoadingHistory] = useState(true);
  const [searchTerm, setSearchTerm] = useState("");
  const [debouncedSearchTerm, setDebouncedSearchTerm] = useState("");
  const [currentPage, setCurrentPage] = useState(1);
  const [itemsPerPage] = useState(10);
  const [selectedAssessmentType, setSelectedAssessmentType] = useState("all");

  // Import CSV state
  const [importAssessmentType, setImportAssessmentType] = useState("Aptitude Round");
  const [importLoading, setImportLoading] = useState(false);
  const fileInputRef = useRef(null);

  const token = localStorage.getItem("adminToken");

  // Debounce search
  useEffect(() => {
    const timer = setTimeout(() => setDebouncedSearchTerm(searchTerm), 300);
    return () => clearTimeout(timer);
  }, [searchTerm]);

  useEffect(() => {
    fetchInviteHistory();
  }, []);

  useEffect(() => {
    let filtered = inviteHistory;
    if (debouncedSearchTerm.trim()) {
      const lower = debouncedSearchTerm.toLowerCase();
      filtered = filtered.filter(
        (i) =>
          i.name.toLowerCase().includes(lower) ||
          i.email.toLowerCase().includes(lower)
      );
    }
    if (selectedAssessmentType !== "all") {
      filtered = filtered.filter((i) => i.assessmentType === selectedAssessmentType);
    }
    setFilteredHistory(filtered);
    setCurrentPage(1);
  }, [debouncedSearchTerm, selectedAssessmentType, inviteHistory]);

  const fetchInviteHistory = async () => {
    try {
      setIsLoadingHistory(true);
      const res = await fetch(API_URL_ADMIN_GET_INVITES, {
        method: "GET",
        headers: {
          "Content-Type": "application/json",
          Accept: "application/json",
          Authorization: "Bearer " + token,
        },
      });
      if (res.ok) {
        const data = await res.json();
        const transformed = transformMongoDBData(data.invites || []);
        setInviteHistory(transformed);
        setFilteredHistory(transformed);
      } else {
        loadFromLocalStorage();
      }
    } catch (err) {
      loadFromLocalStorage();
    } finally {
      setIsLoadingHistory(false);
    }
  };

  const loadFromLocalStorage = () => {
    try {
      const saved = localStorage.getItem("adminInviteHistory");
      if (saved) {
        const parsed = JSON.parse(saved);
        setInviteHistory(parsed);
        setFilteredHistory(parsed);
      }
    } catch (error) {
      console.error("Error loading from localStorage:", error);
    }
  };

  const saveToLocalStorage = (history) => {
    try {
      localStorage.setItem("adminInviteHistory", JSON.stringify(history.slice(0, 50)));
    } catch (error) {
      console.error("Error saving to localStorage:", error);
    }
  };

  const transformMongoDBData = (mongoData) => {
    const history = [];
    mongoData.forEach((invite) => {
      if (invite.aptitude_key) {
        history.push({
          id: invite._id + "_aptitude",
          email: invite.candidate_email,
          name: invite.candidate_name || invite.candidate_email.split("@")[0],
          assessmentType: "Aptitude Round",
          timestamp: invite.aptitude_expire
            ? new Date(invite.aptitude_expire).toISOString()
            : invite.created_at
            ? new Date(invite.created_at).toISOString()
            : new Date().toISOString(),
          inviteToken: invite.aptitude_key,
          status: "sent",
          source: "mongodb",
        });
      }
      if (invite.coding_key) {
        history.push({
          id: invite._id + "_coding",
          email: invite.candidate_email,
          name: invite.candidate_name || invite.candidate_email.split("@")[0],
          assessmentType: "Coding Round",
          timestamp: invite.coding_expire
            ? new Date(invite.coding_expire).toISOString()
            : invite.created_at
            ? new Date(invite.created_at).toISOString()
            : new Date().toISOString(),
          inviteToken: invite.coding_key,
          status: "sent",
          source: "mongodb",
        });
      }
    });
    return history.sort(function (a, b) {
      return new Date(b.timestamp) - new Date(a.timestamp);
    });
  };

  const validateEmail = (email) => {
    const re = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return re.test(email);
  };

  const isEmailAlreadyInvited = (email, type) => {
    const lower = email.toLowerCase();
    return inviteHistory.some(
      (i) => i.email.toLowerCase() === lower && i.assessmentType === type
    );
  };

  const getEmailInviteHistory = (email) => {
    const lower = email.toLowerCase();
    return inviteHistory
      .filter((i) => i.email.toLowerCase() === lower)
      .sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
  };

  const adminSendInvite = async () => {
    if (!adminCandidateEmail || !adminCandidateName || !validateEmail(adminCandidateEmail)) {
      alert("Enter valid name & email");
      return;
    }

    if (isEmailAlreadyInvited(adminCandidateEmail, adminSelectedPosition)) {
      const lastList = getEmailInviteHistory(adminCandidateEmail);
      const last = lastList.length > 0 ? lastList[0] : null;
      if (last) {
        setPendingInvite({
          email: adminCandidateEmail,
          name: adminCandidateName,
          assessmentType: adminSelectedPosition,
          lastSent: last.timestamp,
        });
        setShowResendConfirm(true);
        return;
      }
    }

    await sendInviteRequest();
  };

  const sendInviteRequest = async () => {
    setLoading(true);
    try {
      const res = await fetch(API_URL_ADMIN_SEND_INVITE_LINK, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Accept: "application/json",
          Authorization: "Bearer " + token,
        },
        body: JSON.stringify({
          email: adminCandidateEmail,
          assessment_type: adminSelectedPosition,
          candidate_name: adminCandidateName,
        }),
      });

      const data = await res.json();

      if (res.ok) {
        alert("✅ Invite sent to " + adminCandidateName + " (" + adminCandidateEmail + ")!");
        setAdminCandidateEmail("");
        setAdminCandidateName("");
        await fetchInviteHistory();
      } else {
        alert("❌ Error Sending Mail: " + (data.detail || "Unknown error"));
      }
    } catch (e) {
      alert("❌ Error: " + (e.message || "Unknown error"));
    } finally {
      setLoading(false);
      setShowResendConfirm(false);
      setPendingInvite(null);
    }
  };

  const handleResendConfirm = () => {
    sendInviteRequest();
  };

  const handleResendCancel = () => {
    setShowResendConfirm(false);
    setPendingInvite(null);
  };

  const handleResendInvite = async (invite) => {
    const confirmMsg =
      "Resend invite to " +
      invite.name +
      " (" +
      invite.email +
      ") for " +
      invite.assessmentType +
      "?";

    if (!window.confirm(confirmMsg)) {
      return;
    }

    setLoading(true);
    try {
      const res = await fetch(API_URL_ADMIN_SEND_INVITE_LINK, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Accept: "application/json",
          Authorization: "Bearer " + token,
        },
        body: JSON.stringify({
          email: invite.email,
          assessment_type: invite.assessmentType,
          candidate_name: invite.name,
        }),
      });
      const data = await res.json();

      if (res.ok) {
        alert("✅ Invite resent to " + invite.name + " (" + invite.email + ")!");
        await fetchInviteHistory();
      } else {
        alert("❌ Error Sending Mail: " + (data.detail || "Unknown error"));
      }
    } catch (e) {
      alert("❌ Error: " + (e.message || "Unknown error"));
    } finally {
      setLoading(false);
    }
  };

  const clearHistory = () => {
    if (window.confirm("Clear all invitation history?")) {
      localStorage.removeItem("adminInviteHistory");
      setInviteHistory([]);
      setFilteredHistory([]);
      alert("Local history cleared!");
    }
  };

  const exportToCSV = () => {
    const headers = ["Name", "Email", "Assessment Type", "Invite Code", "Sent Date"];
    const csvData = filteredHistory.map(function (i) {
      return [
        '"' + i.name + '"',
        '"' + i.email + '"',
        '"' + i.assessmentType + '"',
        '"' + i.inviteToken + '"',
        '"' + formatDate(i.timestamp) + '"',
      ];
    });
    const csvContent = [headers]
      .concat(csvData)
      .map(function (r) {
        return r.join(",");
      })
      .join("\n");

    const blob = new Blob([csvContent], { type: "text/csv" });
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = "invites-" + new Date().toISOString().split("T")[0] + ".csv";
    link.click();
    window.URL.revokeObjectURL(url);
  };

  const formatDate = (d) =>
    new Date(d).toLocaleString("en-US", {
      year: "numeric",
      month: "short",
      day: "numeric",
      hour: "2-digit",
      minute: "2-digit",
    });

  const getTimeAgo = (d) => {
    const diff = (new Date() - new Date(d)) / 60000;
    if (diff < 1) return "Just now";
    if (diff < 60) return Math.floor(diff) + "m ago";
    if (diff / 60 < 24) return Math.floor(diff / 60) + "h ago";
    return Math.floor(diff / 1440) + "d ago";
  };

  // Pagination
  const indexOfLastItem = currentPage * itemsPerPage;
  const indexOfFirstItem = indexOfLastItem - itemsPerPage;
  const currentItems = filteredHistory.slice(indexOfFirstItem, indexOfLastItem);
  const totalPages = Math.ceil(filteredHistory.length / itemsPerPage);
  const nextPage = () => {
    if (currentPage < totalPages) setCurrentPage(currentPage + 1);
  };
  const prevPage = () => {
    if (currentPage > 1) setCurrentPage(currentPage - 1);
  };

  // ----- CSV IMPORT HELPERS -----

  const parseCSVContent = (text) => {
    const lines = text.split(/\r?\n/).filter(function (l) {
      return l.trim() !== "";
    });
    if (lines.length === 0) return [];

    const header = lines[0]
      .split(",")
      .map(function (h) {
        return h.trim().toLowerCase();
      });

    const nameIndex = header.findIndex(function (h) {
      return h === "name" || h === "candidate_name";
    });
    const emailIndex = header.findIndex(function (h) {
      return h === "email" || h === "candidate_email";
    });

    const rows = [];
    for (let i = 1; i < lines.length; i++) {
      const cols = lines[i].split(",");
      if (emailIndex === -1 || !cols[emailIndex]) continue;

      const email = cols[emailIndex].trim();
      const name =
        nameIndex !== -1 && cols[nameIndex]
          ? cols[nameIndex].trim()
          : email.split("@")[0];

      rows.push({ name: name, email: email });
    }
    return rows;
  };

  const handleImportClick = () => {
    if (fileInputRef.current) {
      fileInputRef.current.value = "";
      fileInputRef.current.click();
    }
  };

  const handleCSVUpload = async (e) => {
    const files = e.target.files;
    const file = files && files[0];
    if (!file) return;

    try {
      const text = await file.text();
      const rows = parseCSVContent(text);

      if (!rows.length) {
        alert("CSV file seems empty or invalid.");
        return;
      }

      const confirmMsg =
        "Send " + rows.length + " invites for " + importAssessmentType + "?";
      if (!window.confirm(confirmMsg)) {
        return;
      }

      setImportLoading(true);

      let success = 0;
      let failed = 0;
      let skipped = 0;

      for (let i = 0; i < rows.length; i++) {
        const row = rows[i];
        const email = row.email;
        const name = row.name || row.email.split("@")[0];

        if (!validateEmail(email)) {
          failed++;
          continue;
        }

        if (isEmailAlreadyInvited(email, importAssessmentType)) {
          skipped++;
          continue;
        }

        try {
          const res = await fetch(API_URL_ADMIN_SEND_INVITE_LINK, {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
              Accept: "application/json",
              Authorization: "Bearer " + token,
            },
            body: JSON.stringify({
              email: email,
              assessment_type: importAssessmentType,
              candidate_name: name,
            }),
          });

          let data = {};
          try {
            data = await res.json();
          } catch (err) {
            data = {};
          }

          if (res.ok) {
            success++;
          } else {
            console.error("Invite error:", data);
            failed++;
          }
        } catch (err) {
          console.error("Invite error:", err);
          failed++;
        }
      }

      await fetchInviteHistory();

      alert(
        "Import finished for " +
          importAssessmentType +
          ".\n\n" +
          "✅ Sent: " +
          success +
          "\n" +
          "⏭️ Skipped (already invited): " +
          skipped +
          "\n" +
          "❌ Failed: " +
          failed
      );
    } catch (err) {
      console.error("CSV import error:", err);
      alert("Error reading CSV file.");
    } finally {
      setImportLoading(false);
    }
  };

  // -----------------------------

  const adminStyles = {
    adminSection: {
      background: "#fff",
      borderRadius: "8px",
      boxShadow: "0 2px 6px rgba(0,0,0,0.1)",
      border: "1px solid #e2e8f0",
    },
    adminSectionHeader: {
      padding: "1.25rem 1.5rem",
      borderBottom: "1px solid #e2e8f0",
      display: "flex",
      justifyContent: "space-between",
      alignItems: "center",
    },
    adminSectionTitle: {
      fontSize: "1.25rem",
      fontWeight: "600",
      color: "#1a202c",
      margin: 0,
      display: "flex",
      alignItems: "center",
      gap: "0.5rem",
    },
    adminFormGroup: {
      display: "flex",
      flexDirection: "column",
      gap: "0.5rem",
    },
    adminLabel: {
      fontWeight: "500",
      color: "#374151",
      fontSize: "0.875rem",
    },
    adminInput: {
      padding: "0.5rem 0.75rem",
      border: "1px solid #d1d5db",
      borderRadius: "4px",
      fontSize: "0.875rem",
      transition: "border-color 0.2s",
    },
    adminSelect: {
      padding: "0.5rem 0.75rem",
      border: "1px solid #d1d5db",
      borderRadius: "4px",
      fontSize: "0.875rem",
    },
    adminBtn: {
      border: "none",
      borderRadius: "6px",
      padding: "0.5rem 1rem",
      fontWeight: "500",
      cursor: "pointer",
      transition: "all 0.2s",
      display: "inline-flex",
      alignItems: "center",
      gap: "0.4rem",
      fontSize: "0.875rem",
    },
    adminBtnPrimary: {
      background: "#3b82f6",
      color: "white",
    },
    searchContainer: {
      position: "relative",
      marginBottom: "1rem",
    },
    searchInput: {
      width: "100%",
      padding: "0.5rem 2.5rem 0.5rem 0.75rem",
      border: "1px solid #d1d5db",
      borderRadius: "4px",
      fontSize: "0.875rem",
    },
    searchIcon: {
      position: "absolute",
      right: "0.75rem",
      top: "50%",
      transform: "translateY(-50%)",
      color: "#6b7280",
    },
    filterSelect: {
      padding: "0.5rem 0.75rem",
      border: "1px solid #d1d5db",
      borderRadius: "4px",
      fontSize: "0.875rem",
      background: "white",
    },
    pagination: {
      display: "flex",
      justifyContent: "space-between",
      alignItems: "center",
      marginTop: "1rem",
      padding: "1rem 0",
      borderTop: "1px solid #e2e8f0",
    },
    paginationInfo: {
      color: "#6b7280",
      fontSize: "0.875rem",
    },
    paginationControls: {
      display: "flex",
      alignItems: "center",
      gap: "0.5rem",
    },
    paginationButton: {
      padding: "0.5rem 0.75rem",
      border: "1px solid #d1d5db",
      background: "white",
      borderRadius: "4px",
      cursor: "pointer",
      fontSize: "0.875rem",
      display: "flex",
      alignItems: "center",
      gap: "0.25rem",
    },
    paginationButtonDisabled: {
      opacity: 0.5,
      cursor: "not-allowed",
    },
    currentPage: {
      padding: "0.5rem 0.75rem",
      background: "#3b82f6",
      color: "white",
      borderRadius: "4px",
      fontSize: "0.875rem",
      fontWeight: "500",
    },
    adminTable: {
      width: "100%",
      borderCollapse: "collapse",
    },
    adminTableHeader: {
      background: "#f8fafc",
    },
    adminTableCell: {
      padding: "0.75rem",
      textAlign: "left",
      borderBottom: "1px solid #e2e8f0",
      fontSize: "0.875rem",
    },
    adminCandidateInfo: {
      display: "flex",
      alignItems: "center",
      gap: "0.75rem",
    },
    adminStatus: {
      padding: "0.25rem 0.5rem",
      borderRadius: "4px",
      fontSize: "0.75rem",
      fontWeight: "500",
      display: "inline-flex",
      alignItems: "center",
      gap: "0.25rem",
    },
    resendBtn: {
      padding: "0.25rem 0.5rem",
      background: "#fbbf24",
      borderRadius: "4px",
      cursor: "pointer",
      fontSize: "0.75rem",
      fontWeight: "500",
      display: "inline-flex",
      alignItems: "center",
      gap: "0.25rem",
      border: "none",
    },
  };

  return (
    <>
      {/* Resend Confirmation Modal */}
      {showResendConfirm && pendingInvite && (
        <div
          style={{
            position: "fixed",
            top: 0,
            left: 0,
            right: 0,
            bottom: 0,
            background: "rgba(0,0,0,0.5)",
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            zIndex: 1000,
          }}
        >
          <div
            style={{
              background: "white",
              padding: "2rem",
              borderRadius: "8px",
              maxWidth: "500px",
              width: "90%",
            }}
          >
            <div
              style={{
                display: "flex",
                alignItems: "center",
                gap: "0.5rem",
                marginBottom: "1rem",
              }}
            >
              <AlertCircle size={24} color="#f59e0b" />
              <h3 style={{ margin: 0, color: "#1a202c" }}>Resend Invite?</h3>
            </div>
            <p style={{ marginBottom: "1.5rem", color: "#6b7280" }}>
              Already invited for{" "}
              <strong>{pendingInvite.assessmentType}</strong>. Last sent{" "}
              {getTimeAgo(pendingInvite.lastSent)}.
            </p>
            <div
              style={{
                background: "#fffbeb",
                padding: "1rem",
                borderRadius: "4px",
                marginBottom: "1.5rem",
              }}
            >
              <div style={{ fontWeight: "600", marginBottom: "0.25rem" }}>
                {pendingInvite.name}
              </div>
              <div style={{ color: "#6b7280", fontSize: "0.875rem" }}>
                {pendingInvite.email}
              </div>
              <div
                style={{
                  color: "#d97706",
                  fontSize: "0.75rem",
                  marginTop: "0.5rem",
                }}
              >
                Last sent: {formatDate(pendingInvite.lastSent)}
              </div>
            </div>
            <div
              style={{ display: "flex", gap: "0.75rem", justifyContent: "flex-end" }}
            >
              <button
                style={{
                  ...adminStyles.adminBtn,
                  background: "#6b7280",
                  color: "white",
                }}
                onClick={handleResendCancel}
                disabled={loading}
              >
                Cancel
              </button>
              <button
                style={{
                  ...adminStyles.adminBtn,
                  background: "#3b82f6",
                  color: "white",
                }}
                onClick={handleResendConfirm}
                disabled={loading}
              >
                {loading ? "Sending..." : "Resend Invite"}
              </button>
            </div>
          </div>
        </div>
      )}

      {/* Main Component */}
      <section style={adminStyles.adminSection}>
        <div style={adminStyles.adminSectionHeader}>
          <div>
            <h2 style={adminStyles.adminSectionTitle}>
              <Users size={20} /> Candidate Management
            </h2>
            <p
              style={{
                margin: "0.25rem 0 0 0",
                color: "#6b7280",
                fontSize: "0.875rem",
              }}
            >
              Send assessment invites and track invitation history
            </p>
          </div>
          <div style={{ display: "flex", gap: "0.5rem", alignItems: "center" }}>
            {/* Import round selection */}
            <div style={{ display: "flex", alignItems: "center", gap: "0.5rem" }}>
              <span style={{ fontSize: "0.8rem", color: "#6b7280" }}>
                Import Round:
              </span>
              <select
                value={importAssessmentType}
                onChange={function (e) {
                  setImportAssessmentType(e.target.value);
                }}
                style={adminStyles.filterSelect}
              >
                <option value="Aptitude Round">Aptitude Round</option>
                <option value="Coding Round">Coding Round</option>
              </select>
            </div>

            {/* Import CSV Button */}
            <button
              style={{
                ...adminStyles.adminBtn,
                background: "#0ea5e9",
                color: "white",
                opacity: importLoading ? 0.7 : 1,
              }}
              onClick={handleImportClick}
              disabled={importLoading}
            >
              <Upload size={14} /> {importLoading ? "Importing..." : "Import CSV"}
            </button>

            {/* Export / Refresh / Clear */}
            <button
              style={{ ...adminStyles.adminBtn, background: "#10b981", color: "white" }}
              onClick={exportToCSV}
              disabled={filteredHistory.length === 0}
            >
              <Download size={14} /> Export CSV
            </button>
            <button
              style={{ ...adminStyles.adminBtn, background: "#3b82f6", color: "white" }}
              onClick={fetchInviteHistory}
              disabled={isLoadingHistory}
            >
              <RefreshCw size={14} /> {isLoadingHistory ? "Loading..." : "Refresh"}
            </button>
            {inviteHistory.length > 0 && (
              <button
                style={{ ...adminStyles.adminBtn, background: "#ef4444", color: "white" }}
                onClick={clearHistory}
                title="Clear local history"
              >
                <Trash2 size={14} /> Clear Local
              </button>
            )}
          </div>
        </div>

        {/* Hidden file input for CSV */}
        <input
          type="file"
          accept=".csv"
          ref={fileInputRef}
          style={{ display: "none" }}
          onChange={handleCSVUpload}
        />

        {/* Candidate Form */}
        <div
          style={{
            padding: "1.25rem 1.5rem",
            borderBottom: "1px solid #e2e8f0",
            display: "grid",
            gridTemplateColumns: "1fr 1fr 1fr auto",
            gap: "1rem",
            alignItems: "end",
          }}
        >
          <div style={adminStyles.adminFormGroup}>
            <label style={adminStyles.adminLabel}>Candidate Name</label>
            <input
              type="text"
              style={adminStyles.adminInput}
              placeholder="Enter candidate name"
              value={adminCandidateName}
              onChange={function (e) {
                setAdminCandidateName(e.target.value);
              }}
            />
          </div>
          <div style={adminStyles.adminFormGroup}>
            <label style={adminStyles.adminLabel}>Candidate Email</label>
            <input
              type="email"
              style={adminStyles.adminInput}
              placeholder="Enter candidate email"
              value={adminCandidateEmail}
              onChange={function (e) {
                setAdminCandidateEmail(e.target.value);
              }}
            />
          </div>
          <div style={adminStyles.adminFormGroup}>
            <label style={adminStyles.adminLabel}>Assessment Type</label>
            <select
              style={adminStyles.adminSelect}
              value={adminSelectedPosition}
              onChange={function (e) {
                setAdminSelectedPosition(e.target.value);
              }}
            >
              <option value="Aptitude Round">Aptitude Round</option>
              <option value="Coding Round">Coding Round</option>
            </select>
          </div>
          <button
            style={{
              ...adminStyles.adminBtn,
              ...adminStyles.adminBtnPrimary,
              opacity: loading ? 0.6 : 1,
            }}
            onClick={adminSendInvite}
            disabled={loading}
          >
            {loading ? (
              <>
                <Clock size={16} /> Sending...
              </>
            ) : (
              <>
                <Send size={16} /> Send Invite
              </>
            )}
          </button>
        </div>

        {/* Invitation History */}
        <div style={{ padding: "1.5rem" }}>
          <div
            style={{
              display: "flex",
              justifyContent: "space-between",
              alignItems: "center",
              marginBottom: "1rem",
            }}
          >
            <h3
              style={{
                fontSize: "1rem",
                fontWeight: "600",
                color: "#1a202c",
                margin: 0,
                display: "flex",
                alignItems: "center",
                gap: "0.5rem",
              }}
            >
              <Mail size={16} /> Invitation History ({filteredHistory.length})
              {debouncedSearchTerm && (
                <span
                  style={{
                    fontSize: "0.875rem",
                    fontWeight: "normal",
                    color: "#6b7280",
                    marginLeft: "0.5rem",
                  }}
                >
                  (Filtered by "{debouncedSearchTerm}")
                </span>
              )}
            </h3>
            <div style={{ display: "flex", gap: "1rem", alignItems: "center" }}>
              <div style={{ display: "flex", alignItems: "center", gap: "0.5rem" }}>
                <Filter size={14} color="#6b7280" />
                <select
                  value={selectedAssessmentType}
                  onChange={function (e) {
                    setSelectedAssessmentType(e.target.value);
                  }}
                  style={adminStyles.filterSelect}
                >
                  <option value="all">All Types</option>
                  <option value="Aptitude Round">Aptitude</option>
                  <option value="Coding Round">Coding</option>
                </select>
              </div>
              <div style={adminStyles.searchContainer}>
                <input
                  type="text"
                  placeholder="Search by name or email..."
                  value={searchTerm}
                  onChange={function (e) {
                    setSearchTerm(e.target.value);
                  }}
                  style={adminStyles.searchInput}
                />
                <Search size={16} style={adminStyles.searchIcon} />
              </div>
            </div>
          </div>

          {isLoadingHistory ? (
            <div style={{ textAlign: "center", padding: "2rem", color: "#6b7280" }}>
              <RefreshCw
                size={24}
                style={{
                  marginBottom: "0.5rem",
                  animation: "spin 1s linear infinite",
                }}
              />
              <p>Loading invitation history from database...</p>
            </div>
          ) : filteredHistory.length === 0 ? (
            <div style={{ textAlign: "center", padding: "3rem", color: "#6b7280" }}>
              <Mail size={48} style={{ marginBottom: "1rem", opacity: 0.5 }} />
              <p>
                {searchTerm || selectedAssessmentType !== "all"
                  ? "No invitations found matching your filters"
                  : "No invitations found"}
              </p>
              <p style={{ fontSize: "0.875rem", marginTop: "0.5rem" }}>
                {searchTerm || selectedAssessmentType !== "all"
                  ? "Try adjusting your search or filters"
                  : "Send your first invite to see it here"}
              </p>
            </div>
          ) : (
            <>
              <table style={adminStyles.adminTable}>
                <thead style={adminStyles.adminTableHeader}>
                  <tr>
                    <th style={adminStyles.adminTableCell}>Candidate</th>
                    <th style={adminStyles.adminTableCell}>Assessment Type</th>
                    <th style={adminStyles.adminTableCell}>Invite Code</th>
                    <th style={adminStyles.adminTableCell}>Sent Date</th>
                    <th style={adminStyles.adminTableCell}>Status</th>
                    <th style={adminStyles.adminTableCell}>Actions</th>
                  </tr>
                </thead>
                <tbody>
                  {currentItems.map(function (invite) {
                    return (
                      <tr key={invite.id}>
                        <td style={adminStyles.adminTableCell}>
                          <div style={adminStyles.adminCandidateInfo}>
                            <div
                              style={{
                                width: "32px",
                                height: "32px",
                                borderRadius: "50%",
                                background: "#3b82f6",
                                display: "flex",
                                alignItems: "center",
                                justifyContent: "center",
                                color: "white",
                                fontWeight: "600",
                                fontSize: "0.875rem",
                              }}
                            >
                              {invite.name.charAt(0).toUpperCase()}
                            </div>
                            <div>
                              <div
                                style={{ fontWeight: "600", color: "#1a202c" }}
                              >
                                {invite.name}
                              </div>
                              <div
                                style={{
                                  fontSize: "0.75rem",
                                  color: "#64748b",
                                }}
                              >
                                {invite.email}
                              </div>
                              <div
                                style={{
                                  fontSize: "0.7rem",
                                  color: "#94a3b8",
                                  marginTop: "0.125rem",
                                }}
                              >
                                {invite.source === "mongodb"
                                  ? "From Database"
                                  : "New Invite"}
                              </div>
                            </div>
                          </div>
                        </td>
                        <td style={adminStyles.adminTableCell}>
                          {invite.assessmentType}
                        </td>
                        <td style={adminStyles.adminTableCell}>
                          <code
                            style={{
                              background: "#f3f4f6",
                              padding: "0.25rem 0.5rem",
                              borderRadius: "4px",
                              fontSize: "0.75rem",
                              fontFamily: "monospace",
                              color: "#374151",
                            }}
                          >
                            {invite.inviteToken}
                          </code>
                        </td>
                        <td style={adminStyles.adminTableCell}>
                          <div style={{ fontSize: "0.875rem" }}>
                            {formatDate(invite.timestamp)}
                          </div>
                          <div
                            style={{
                              fontSize: "0.75rem",
                              color: "#64748b",
                            }}
                          >
                            {getTimeAgo(invite.timestamp)}
                          </div>
                        </td>
                        <td style={adminStyles.adminTableCell}>
                          <span
                            style={{
                              ...adminStyles.adminStatus,
                              background: "#f0fdf4",
                              color: "#166534",
                            }}
                          >
                            <CheckCircle
                              size={12}
                              style={{ marginRight: "0.25rem" }}
                            />
                            Sent
                          </span>
                        </td>
                        <td style={adminStyles.adminTableCell}>
                          <button
                            style={adminStyles.resendBtn}
                            onClick={function () {
                              handleResendInvite(invite);
                            }}
                            disabled={loading}
                          >
                            <Repeat size={12} /> Resend
                          </button>
                        </td>
                      </tr>
                    );
                  })}
                </tbody>
              </table>

              {totalPages > 1 && (
                <div style={adminStyles.pagination}>
                  <div style={adminStyles.paginationInfo}>
                    Showing {indexOfFirstItem + 1}-
                    {Math.min(indexOfLastItem, filteredHistory.length)} of{" "}
                    {filteredHistory.length} invitations
                  </div>
                  <div style={adminStyles.paginationControls}>
                    <button
                      style={{
                        ...adminStyles.paginationButton,
                        ...(currentPage === 1
                          ? adminStyles.paginationButtonDisabled
                          : {}),
                      }}
                      onClick={prevPage}
                      disabled={currentPage === 1}
                    >
                      <ChevronLeft size={16} /> Previous
                    </button>
                    <div style={adminStyles.currentPage}>
                      {currentPage} / {totalPages}
                    </div>
                    <button
                      style={{
                        ...adminStyles.paginationButton,
                        ...(currentPage === totalPages
                          ? adminStyles.paginationButtonDisabled
                          : {}),
                      }}
                      onClick={nextPage}
                      disabled={currentPage === totalPages}
                    >
                      Next <ChevronRight size={16} />
                    </button>
                  </div>
                </div>
              )}
            </>
          )}
        </div>
      </section>
      <style>{`@keyframes spin {from {transform: rotate(0deg);} to {transform: rotate(360deg);}}`}</style>
    </>
  );
};

export default AdminCandidateManagement;
