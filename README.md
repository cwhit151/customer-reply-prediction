import streamlit as st
import requests
import time
import random

st.set_page_config(layout="wide")

# Databricks endpoint URL
ENDPOINT_URL = "https://dbc-624143a1-5376.cloud.databricks.com/serving-endpoints/Final_Project/invocations"

# Databricks personal access token
DATABRICKS_TOKEN = "dapi..."

headers = {
    "Authorization": f"Bearer {DATABRICKS_TOKEN}",
    "Content-Type": "application/json"
}




# ----------------------------------------------------
# Custom CSS (FULL UI UPGRADE)
# ----------------------------------------------------
st.markdown(
    """
<style>

html, body, .block-container {
    background-color: #0e0e0e !important;
}

/* Neon Animated Title */
.neon-title {
    font-size: 45px;
    text-align: center;
    color: #ff4b4b;
    font-weight: 800;
    text-shadow:
        0 0 5px #ff4b4b,
        0 0 10px #ff4b4b,
        0 0 20px #c70000;
    animation: pulse 2s infinite alternate;
}

@keyframes pulse {
    0% { text-shadow: 0 0 8px #ff4b4b; }
    100% { text-shadow: 0 0 20px #ff1a1a; }
}

/* Glass Card */
.glass-card {
    background: rgba(255, 255, 255, 0.08);
    padding: 22px;
    border-radius: 12px;
    border: 1px solid rgba(255,255,255,0.15);
    backdrop-filter: blur(10px);
    color: white;
    margin-top: 20px;
}

/* Center Predict Button container */
.predict-container {
    display: flex;
    justify-content: center;
    margin-top: 25px;
}

/* Neon Button (all st.button) */
div.stButton > button:first-child {
    background: linear-gradient(90deg, #ff3b3b, #ff7b7b);
    color: white;
    font-size: 18px;
    padding: 12px 26px;
    border-radius: 8px;
    width: 200px;
    border: none;
    transition: all 0.2s ease-in-out;
}
div.stButton > button:first-child:hover {
    transform: scale(1.07);
    box-shadow: 0 0 20px #ff4b4b;
}

/* Loader spinner */
.loader {
  border: 6px solid #333;
  border-top: 6px solid #ff5757;
  border-radius: 50%;
  width: 55px;
  height: 55px;
  animation: spin 0.8s linear infinite;
  margin: auto;
  margin-top: 15px;
}
@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

/* Result Cards */
.result-card {
    padding: 25px;
    border-radius: 12px;
    color: white;
    font-size: 24px;
    text-align: center;
    margin-top: 20px;
}
.success-card {
    background: linear-gradient(90deg, #2ecc71, #1e8f4a);
}
.error-card {
    background: linear-gradient(90deg, #e74c3c, #b82514);
}

</style>
""",
    unsafe_allow_html=True,
)

# ----------------------------------------------------
# Helper function — local renewal probability
# ----------------------------------------------------
def calculate_local_probability(features):
    """Generate a pseudo-probability score based on user inputs."""
    score = 0

    # Positive factors
    if features["is_current_customer"] == 1:
        score += 20
    if features["tenure_months"] > 12:
        score += 15
    if features["emails_opened_last_30d"] > 5:
        score += 15
    if features["past_positive_replies"] > 0:
        score += 10
    if features["tag_high_priority"] == 1:
        score += 10

    # Negative factors
    if features["avg_response_time_hours"] > 24:
        score -= 15
    if features["last_interaction_days_ago"] > 20:
        score -= 20
    if features["total_tickets_last_6mo"] > 5:
        score -= 10

    # Keep in range 5–95%
    score = max(5, min(score, 95))
    return score


# ----------------------------------------------------
# Tabs
# ----------------------------------------------------
tab1, tab2 = st.tabs(["🔮 Prediction App", "📘 About the Model"])


# ----------------------------------------------------
# TAB 1 — PREDICTION APP
# ----------------------------------------------------
with tab1:

    # Neon title
    st.markdown(
        "<div class='neon-title'>Customer Renewal Prediction</div>",
        unsafe_allow_html=True,
    )

    # Sidebar inputs
    st.sidebar.header("📊 Customer Inputs")

    industry = st.sidebar.selectbox(
        "Industry", ["Construction", "Healthcare", "Retail", "Finance", "Tech"]
    )
    region = st.sidebar.selectbox(
        "Region", ["South", "West", "Midwest", "Northeast"]
    )
    channel = st.sidebar.selectbox("Channel", ["email", "sms"])
    company_size = st.sidebar.selectbox(
        "Company size", ["Small", "Medium", "Enterprise"]
    )
    tenure_months = st.sidebar.number_input("Tenure (months)", 0, 200, 12)

    is_current_label = st.sidebar.selectbox("Current customer?", ["Yes", "No"])
    is_current_customer = 1 if is_current_label == "Yes" else 0

    total_tickets_last_6mo = st.sidebar.number_input(
        "Tickets (last 6 months)", 0, 50, 2
    )
    avg_response_time_hours = st.sidebar.number_input(
        "Avg response time (hours)", 0.0, 100.0, 3.5
    )

    emails_sent_last_30d = st.sidebar.number_input(
        "Emails sent (30 days)", 0, 100, 15
    )
    emails_opened_last_30d = st.sidebar.number_input(
        "Emails opened (30 days)", 0, 100, 10
    )
    emails_clicked_last_30d = st.sidebar.number_input(
        "Emails clicked (30 days)", 0, 100, 3
    )

    past_positive_replies = st.sidebar.number_input(
        "Past positive replies", 0, 20, 1
    )
    last_interaction_days_ago = st.sidebar.number_input(
        "Days since last interaction", 0, 36, 11
    )

    tag_high_label = st.sidebar.selectbox("High priority tag?", ["Yes", "No"])
    tag_high_priority = 1 if tag_high_label == "Yes" else 0

    tag_new_lead_label = st.sidebar.selectbox("New lead tag?", ["Yes", "No"])
    tag_new_lead = 1 if tag_new_lead_label == "Yes" else 0

    # Build payload for Databricks model
    payload = {
        "dataframe_records": [
            {
                "contact_id": 1,
                "industry": industry,
                "company_size": company_size,
                "region": region,
                "tenure_months": tenure_months,
                "is_current_customer": is_current_customer,
                "total_tickets_last_6mo": total_tickets_last_6mo,
                "avg_response_time_hours": avg_response_time_hours,
                "emails_sent_last_30d": emails_sent_last_30d,
                "emails_opened_last_30d": emails_opened_last_30d,
                "emails_clicked_last_30d": emails_clicked_last_30d,
                "past_positive_replies": past_positive_replies,
                "last_interaction_days_ago": last_interaction_days_ago,
                "tag_high_priority": tag_high_priority,
                "tag_new_lead": tag_new_lead,
                "channel": channel,
            }
        ]
    }

    # --------------------------------------
    # SINGLE CENTERED PREDICT BUTTON
    # --------------------------------------
    st.markdown("<div class='predict-container'>", unsafe_allow_html=True)
    clicked = st.button("Predict", key="predict_button")
    st.markdown("</div>", unsafe_allow_html=True)

    # --------------------------------------
    # RUN PREDICTION WHEN CLICKED
    # --------------------------------------
    if clicked:

        # Loader
        loader_placeholder = st.empty()
        loader_placeholder.markdown(
            "<div class='loader'></div>", unsafe_allow_html=True
        )
        time.sleep(1.2)

        # Call Databricks endpoint
        try:
            response = requests.post(
                ENDPOINT_URL, headers=headers, json=payload
            )
            response.raise_for_status()
            result = response.json()
        except Exception as e:
            loader_placeholder.empty()
            st.markdown(
                '<div class="result-card error-card">❌ Error calling prediction API.</div>',
                unsafe_allow_html=True,
            )
            st.exception(e)
            st.stop()

        loader_placeholder.empty()

        # Extract prediction
        try:
            pred = result["predictions"][0]
        except Exception:
            st.markdown(
                '<div class="result-card error-card">❌ Error reading prediction.</div>',
                unsafe_allow_html=True,
            )
            st.json(result)
            st.stop()

        # Summary card
        st.markdown(
            f"""
        <div class="glass-card">
            <h3>📄 Customer Summary</h3>
            Industry: <b>{industry}</b><br>
            Region: <b>{region}</b><br>
            Channel: <b>{channel}</b><br>
            Company size: <b>{company_size}</b><br>
            Tenure: <b>{tenure_months} months</b><br>
            Last interaction: <b>{last_interaction_days_ago} days</b><br>
        </div>
        """,
            unsafe_allow_html=True,
        )

        # Compute LOCAL probability
        local_prob = calculate_local_probability(
            {
                "is_current_customer": is_current_customer,
                "tenure_months": tenure_months,
                "emails_opened_last_30d": emails_opened_last_30d,
                "past_positive_replies": past_positive_replies,
                "tag_high_priority": tag_high_priority,
                "avg_response_time_hours": avg_response_time_hours,
                "last_interaction_days_ago": last_interaction_days_ago,
                "total_tickets_last_6mo": total_tickets_last_6mo,
            }
        )

        # SUCCESS / FAILURE message
        if pred == 1:
            st.balloons()
            st.markdown(
                '<div class="result-card success-card">🔥 YES! This customer is LIKELY to renew.</div>',
                unsafe_allow_html=True,
            )
        else:
            st.markdown(
                '<div class="result-card error-card">⚠️ No — Customer is UNLIKELY to renew.</div>',
                unsafe_allow_html=True,
            )

        # Probability bar
        st.markdown(
            f"""
        <div style='margin-top:20px;'>
            <h4 style='color:#ff4b4b;'>Renewal Likelihood: {local_prob}%</h4>
            <div style="
                width:100%;
                background-color:#222;
                border-radius:8px;
                height:18px;
                margin-top:8px;">
                <div style="
                    width:{local_prob}%;
                    background:linear-gradient(90deg,#ff4b4b,#ff7777);
                    height:18px;
                    border-radius:8px;">
                </div>
            </div>
        </div>
        """,
            unsafe_allow_html=True,
        )

        # Confidence level
        if local_prob >= 70:
            confidence = "High Confidence"
        elif local_prob >= 40:
            confidence = "Medium Confidence"
        else:
            confidence = "Low Confidence"

        st.markdown(
            f"<p style='color:gray; font-size:14px;'>Confidence: <b>{confidence}</b></p>",
            unsafe_allow_html=True,
        )

        # Raw JSON Viewer
        with st.expander("📦 Raw Model Response"):
            st.json(result)


# ----------------------------------------------------
# TAB 2 — ABOUT THE MODEL
# ----------------------------------------------------
with tab2:
    st.markdown(
        """
    ## 📘 About This Model
    This predictive model determines whether a customer is likely to **renew their contract**  
    based on 16 behavioral & engagement features:

    - 📧 Email interactions  
    - 👤 Customer status  
    - 🏗️ Industry & company attributes  
    - 🎟️ Support ticket history  
    - 🔥 Lead priority indicators  
    - ⏳ Tenure & last interaction  
    """,
        unsafe_allow_html=True,
  )
# -------------------------------
# MINI GAME FOOTER (WORKING)
# -------------------------------

game_html = """
<!DOCTYPE html>
<html>
<head>
<style>
    body {
        background: transparent;
        color: black;
        text-align: center;
        font-family: Arial;
    }

    #game-box {
        width: 340px;
        height: 130px;
        background: #0d0d0d;
        border: 2px solid #ff4b4b;
        border-radius: 12px;
        position: relative;
        overflow: hidden;
        margin: auto;
        box-shadow: 0 0 15px #ff4b4b;
    }

    #player {
        font-size: 38px;
        position: absolute;
        left: 20px;
        bottom: 10px;
        transition: bottom 0.05s;
    }

    #obstacle {
        font-size: 32px;
        position: absolute;
        left: 320px;
        bottom: 10px;
    }

    #scoreboard {
        margin-top: 6px;
        font-size: 16px;
        color: #ff4b4b;
        font-weight: bold;
        text-shadow: 0 0 8px #ff4b4b;
    }

    #lives {
        margin-top: 4px;
        font-size: 16px;
        color: #ff7b7b;
        font-weight: bold;
    }

    #end-message {
        margin-top: 10px;
        font-size: 18px;
        color: #ff4b4b;
        font-weight: bold;
    }
</style>
</head>

<body tabindex="0">

<div id="game-box">
    <div id="player">🐌</div>
    <div id="obstacle">🚗</div>
</div>

<div id="scoreboard">
    Score: <span id="score">0</span> / 10
</div>

<div id="lives">
    Lives: <span id="lifeCount">3</span>
</div>

<div id="end-message"></div>

<script>
let player = document.getElementById("player");
let obstacle = document.getElementById("obstacle");

let scoreText = document.getElementById("score");
let lifeText = document.getElementById("lifeCount");
let endMessage = document.getElementById("end-message");

let bottom = 10;
let jumping = false;
let score = 0;
let lives = 7;
let gameActive = true;
let gameLoop = null;

// ---- Jump Controls ----
document.body.addEventListener("keydown", function(event) {
    if (!gameActive) return;

    if ((event.code === "Space" || event.code === "ArrowUp") && !jumping) {
        jumping = true;

        let up = setInterval(() => {
            if (bottom < 65) {
                bottom += 5;
                player.style.bottom = bottom + "px";
            } else {
                clearInterval(up);
                let down = setInterval(() => {
                    if (bottom > 10) {
                        bottom -= 5;
                        player.style.bottom = bottom + "px";
                    } else {
                        clearInterval(down);
                        jumping = false;
                    }
                }, 25);
            }
        }, 25);
    }
});

// ---- GAME START ----
function startGame() {
    obstacle.style.left = "320px";

    gameLoop = setInterval(() => {
        if (!gameActive) return clearInterval(gameLoop);

        let obstacleLeft = parseInt(getComputedStyle(obstacle).getPropertyValue("left"));
        let playerBottom = parseInt(getComputedStyle(player).getPropertyValue("bottom"));

        // Move car
        obstacle.style.left = (obstacleLeft - 6) + "px";

        // Passed car ➝ score
        if (obstacleLeft < -30) {
            obstacle.style.left = "320px";
            score++;
            scoreText.innerHTML = score;

            if (score >= 10) {
                endGame(true);
            }
        }

        // Collision
        if (obstacleLeft < 60 && obstacleLeft > 20 && playerBottom < 40) {
            lives--;
            lifeText.innerHTML = lives;

            if (lives <= 0) {
                endGame(false);
            } else {
                obstacle.style.left = "320px";
            }
        }

    }, 30);
}

function endGame(won) {
    gameActive = false;
    clearInterval(gameLoop);

    if (won) {
        endMessage.innerHTML = "🏆 You made it over 10 cars! You win!";
    } else {
        endMessage.innerHTML = "💥 You lost all 7 lives! Game over.";
    }

    // Disable movement visually
    obstacle.style.left = "-9999px";
}

// Start the game once
startGame();
</script>

</body>
</html>
"""

# ================================
# FOOTER GAME (DISCREET VERSION)
# ================================

# Title centered
st.markdown("<h4 style='text-align:center; color:#aaa;'>🎮 Mini Jump Game</h4>", unsafe_allow_html=True)

# Wrapper so we can style only this expander
st.markdown('<div class="footer-expander">', unsafe_allow_html=True)

with st.expander("Open Game"):
    st.components.v1.html(
        game_html,
        height=220,
        scrolling=False   # KEEP THIS
    )

st.markdown('</div>', unsafe_allow_html=True)

# Clean spacing
st.markdown("<br><br>", unsafe_allow_html=True)

# ================================
# CSS to remove white background in expander
# ================================
st.markdown("""
<style>

/* Prevent game expander from glitching */
.footer-expander .streamlit-expanderContent {
    max-height: 260px !important;
    overflow: hidden !important;
}

.footer-expander .streamlit-expanderHeader {
    background-color: #141414 !important;
    color: #ff4b4b !important;
    border: 1px solid #ff4b4b !important;
    border-radius: 8px;
}

.footer-expander .streamlit-expanderContent {
    background-color: #141414 !important;
    border: 1px solid #ff4b4b !important;
    border-top: none !important;
    padding: 10px !important;
    border-radius: 0 0 8px 8px;
}

.footer-expander {
    max-width: 650px;
    margin: auto;
}

</style>
""", unsafe_allow_html=True)

