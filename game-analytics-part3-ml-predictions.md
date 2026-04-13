# Game Analytics Part 3: ML-Powered Game Analytics

**Building AI Models for Level Difficulty Prediction, Player Behavior, and Automated Testing**

This is Part 3 of a 3-part series on building AI-powered game analytics:
- [Part 1: Client-Side Event Tracking](./game-analytics-part1-event-tracking)
- [Part 2: Streaming Pipeline - Kafka to BigQuery](./game-analytics-part2-data-pipeline)
- **Part 3: ML-Powered Game Analytics** (this post)

---

## Overview

With a robust data pipeline in place, we can now build ML models to:

- **Predict level difficulty** before release
- **Identify at-risk players** likely to churn
- **Automate level testing** with AI agents
- **Optimize game balance** through continuous learning

This post covers practical ML applications for game analytics.

---

## ML Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              ML PIPELINE ARCHITECTURE                               │
└─────────────────────────────────────────────────────────────────────────────────────┘

     Feature Store                  Training                      Serving
  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
  │                 │         │                 │         │                 │
  │    BigQuery     │────────▶│   Vertex AI     │────────▶│   Model API     │
  │   (Features)    │         │   Training      │         │   (Endpoints)   │
  │                 │         │                 │         │                 │
  └────────┬────────┘         └────────┬────────┘         └────────┬────────┘
           │                           │                           │
           │                           │                           │
           ▼                           ▼                           ▼
  ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
  │  Level Features │         │  Model Registry │         │  Game Server    │
  │  User Features  │         │  (Versioning)   │         │  Level Editor   │
  │  Session Data   │         │  Experiments    │         │  A/B Testing    │
  └─────────────────┘         └─────────────────┘         └─────────────────┘

                    ┌─────────────────────────────────┐
                    │        Feedback Loop            │
                    │  Predictions → Outcomes → Data  │
                    └─────────────────────────────────┘
```

---

## Use Case 1: Level Difficulty Prediction

### Problem Statement

Before releasing a new level, predict:
- **Pass rate**: What % of players will complete it?
- **Average moves**: How many moves will players need?
- **Frustration score**: How likely are players to quit?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DIFFICULTY PREDICTION WORKFLOW                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐            │
│  │   Level      │     │   Feature    │     │   Predict    │            │
│  │   Design     │────▶│   Extraction │────▶│   Difficulty │            │
│  │   (JSON)     │     │              │     │              │            │
│  └──────────────┘     └──────────────┘     └──────────────┘            │
│                                                   │                     │
│                                                   ▼                     │
│                              ┌─────────────────────────────────┐       │
│                              │  Predicted Pass Rate: 68%       │       │
│                              │  Predicted Avg Moves: 24        │       │
│                              │  Frustration Score: Low         │       │
│                              │                                 │       │
│                              │  ⚠ Warning: Similar to level    │       │
│                              │    #847 which had 45% pass rate │       │
│                              └─────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Feature Engineering

```python
import json
import numpy as np
from dataclasses import dataclass
from typing import List, Dict, Any

@dataclass
class LevelFeatures:
    """Features extracted from level configuration."""

    # Grid structure
    grid_width: int
    grid_height: int
    total_cells: int
    playable_cells: int

    # Objectives
    num_objectives: int
    objective_types: List[str]
    total_target_count: int

    # Available moves
    moves_limit: int
    moves_per_objective: float

    # Blockers and obstacles
    num_blocker_types: int
    total_blockers: int
    blocker_density: float

    # Special elements
    num_special_candies: int
    has_chocolate: bool
    has_licorice: bool
    has_bombs: bool

    # Board complexity
    isolated_regions: int
    conveyor_belts: int
    teleporters: int

    # Historical similar levels
    similar_level_avg_pass_rate: float
    similar_level_avg_moves: float


def extract_features(level_config: Dict[str, Any]) -> LevelFeatures:
    """Extract ML features from level JSON configuration."""

    grid = level_config['grid']
    objectives = level_config['objectives']
    blockers = level_config.get('blockers', [])

    # Grid analysis
    width = len(grid[0])
    height = len(grid)
    total_cells = width * height
    playable_cells = sum(1 for row in grid for cell in row if cell != 'X')

    # Objective analysis
    total_targets = sum(obj.get('count', 0) for obj in objectives)
    objective_types = [obj['type'] for obj in objectives]

    # Blocker analysis
    blocker_types = set(b['type'] for b in blockers)
    total_blockers = sum(b.get('count', 0) for b in blockers)
    blocker_density = total_blockers / playable_cells if playable_cells > 0 else 0

    # Special elements
    special_elements = level_config.get('special_elements', {})

    return LevelFeatures(
        grid_width=width,
        grid_height=height,
        total_cells=total_cells,
        playable_cells=playable_cells,
        num_objectives=len(objectives),
        objective_types=objective_types,
        total_target_count=total_targets,
        moves_limit=level_config.get('moves', 30),
        moves_per_objective=level_config.get('moves', 30) / max(total_targets, 1),
        num_blocker_types=len(blocker_types),
        total_blockers=total_blockers,
        blocker_density=blocker_density,
        num_special_candies=special_elements.get('special_candy_count', 0),
        has_chocolate=special_elements.get('has_chocolate', False),
        has_licorice=special_elements.get('has_licorice', False),
        has_bombs=special_elements.get('has_bombs', False),
        isolated_regions=count_isolated_regions(grid),
        conveyor_belts=len(level_config.get('conveyors', [])),
        teleporters=len(level_config.get('teleporters', [])),
        similar_level_avg_pass_rate=0.0,  # Filled by similarity search
        similar_level_avg_moves=0.0,
    )


def count_isolated_regions(grid: List[List[str]]) -> int:
    """Count disconnected playable regions using flood fill."""
    visited = set()
    regions = 0

    def flood_fill(r, c):
        if (r, c) in visited or r < 0 or c < 0:
            return
        if r >= len(grid) or c >= len(grid[0]):
            return
        if grid[r][c] == 'X':
            return

        visited.add((r, c))
        for dr, dc in [(0, 1), (0, -1), (1, 0), (-1, 0)]:
            flood_fill(r + dr, c + dc)

    for r in range(len(grid)):
        for c in range(len(grid[0])):
            if (r, c) not in visited and grid[r][c] != 'X':
                flood_fill(r, c)
                regions += 1

    return regions
```

### Model Training

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.metrics import mean_absolute_error, r2_score
import joblib
from google.cloud import bigquery

class DifficultyPredictor:
    """Predict level pass rate and average moves."""

    def __init__(self):
        self.pass_rate_model = GradientBoostingRegressor(
            n_estimators=200,
            max_depth=6,
            learning_rate=0.1,
            subsample=0.8,
            random_state=42
        )
        self.moves_model = GradientBoostingRegressor(
            n_estimators=200,
            max_depth=6,
            learning_rate=0.1,
            subsample=0.8,
            random_state=42
        )
        self.feature_columns = None

    def load_training_data(self) -> pd.DataFrame:
        """Load historical level performance data from BigQuery."""
        client = bigquery.Client()

        query = """
        SELECT
            l.level_id,
            l.grid_width,
            l.grid_height,
            l.playable_cells,
            l.num_objectives,
            l.total_target_count,
            l.moves_limit,
            l.num_blocker_types,
            l.total_blockers,
            l.blocker_density,
            l.has_chocolate,
            l.has_licorice,
            l.has_bombs,
            l.isolated_regions,
            l.conveyor_belts,
            l.teleporters,
            m.pass_rate,
            m.avg_moves,
            m.attempts
        FROM `project.analytics.level_features` l
        JOIN `project.analytics.level_metrics` m
            ON l.level_id = m.level_id
        WHERE m.attempts >= 1000  -- Only levels with enough data
          AND m.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
        """

        return client.query(query).to_dataframe()

    def train(self):
        """Train both prediction models."""
        df = self.load_training_data()

        # Define features
        self.feature_columns = [
            'grid_width', 'grid_height', 'playable_cells',
            'num_objectives', 'total_target_count', 'moves_limit',
            'num_blocker_types', 'total_blockers', 'blocker_density',
            'has_chocolate', 'has_licorice', 'has_bombs',
            'isolated_regions', 'conveyor_belts', 'teleporters'
        ]

        X = df[self.feature_columns]
        y_pass_rate = df['pass_rate']
        y_moves = df['avg_moves']

        # Split data
        X_train, X_test, y_pr_train, y_pr_test = train_test_split(
            X, y_pass_rate, test_size=0.2, random_state=42
        )
        _, _, y_mv_train, y_mv_test = train_test_split(
            X, y_moves, test_size=0.2, random_state=42
        )

        # Train pass rate model
        self.pass_rate_model.fit(X_train, y_pr_train)
        pr_pred = self.pass_rate_model.predict(X_test)
        print(f"Pass Rate Model - MAE: {mean_absolute_error(y_pr_test, pr_pred):.3f}")
        print(f"Pass Rate Model - R2: {r2_score(y_pr_test, pr_pred):.3f}")

        # Train moves model
        self.moves_model.fit(X_train, y_mv_train)
        mv_pred = self.moves_model.predict(X_test)
        print(f"Moves Model - MAE: {mean_absolute_error(y_mv_test, mv_pred):.3f}")
        print(f"Moves Model - R2: {r2_score(y_mv_test, mv_pred):.3f}")

        # Feature importance
        self._print_feature_importance()

    def _print_feature_importance(self):
        """Print most important features."""
        importance = pd.DataFrame({
            'feature': self.feature_columns,
            'importance': self.pass_rate_model.feature_importances_
        }).sort_values('importance', ascending=False)

        print("\nTop Features for Pass Rate Prediction:")
        print(importance.head(10).to_string(index=False))

    def predict(self, features: LevelFeatures) -> Dict[str, float]:
        """Predict difficulty metrics for a new level."""
        X = pd.DataFrame([{
            col: getattr(features, col)
            for col in self.feature_columns
        }])

        pass_rate = self.pass_rate_model.predict(X)[0]
        avg_moves = self.moves_model.predict(X)[0]

        # Classify difficulty
        if pass_rate >= 0.75:
            difficulty = "Easy"
        elif pass_rate >= 0.55:
            difficulty = "Medium"
        elif pass_rate >= 0.35:
            difficulty = "Hard"
        else:
            difficulty = "Very Hard"

        return {
            'predicted_pass_rate': round(pass_rate, 3),
            'predicted_avg_moves': round(avg_moves, 1),
            'difficulty_class': difficulty,
            'confidence': self._calculate_confidence(features)
        }

    def _calculate_confidence(self, features: LevelFeatures) -> str:
        """Estimate prediction confidence based on feature similarity."""
        # Check if level is within training distribution
        # (simplified - would use more sophisticated methods in production)
        if features.moves_limit < 10 or features.moves_limit > 100:
            return "Low - unusual move limit"
        if features.isolated_regions > 3:
            return "Medium - complex board layout"
        return "High"

    def save(self, path: str):
        """Save trained models."""
        joblib.dump({
            'pass_rate_model': self.pass_rate_model,
            'moves_model': self.moves_model,
            'feature_columns': self.feature_columns
        }, path)

    @classmethod
    def load(cls, path: str) -> 'DifficultyPredictor':
        """Load trained models."""
        data = joblib.load(path)
        predictor = cls()
        predictor.pass_rate_model = data['pass_rate_model']
        predictor.moves_model = data['moves_model']
        predictor.feature_columns = data['feature_columns']
        return predictor
```

---

## Use Case 2: Player Churn Prediction

### Identifying At-Risk Players

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CHURN PREDICTION PIPELINE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Daily Player Snapshot                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  user_id: abc123                                                  │  │
│  │  days_since_install: 45                                          │  │
│  │  sessions_last_7d: 3  (↓ from 8)                                 │  │
│  │  levels_completed_7d: 5  (↓ from 15)                             │  │
│  │  current_level: 156                                              │  │
│  │  consecutive_fails: 12  (⚠ high)                                 │  │
│  │  purchases_lifetime: $4.99                                       │  │
│  │  days_since_last_session: 2                                      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼                                          │
│                    ┌──────────────────┐                                │
│                    │  Churn Model     │                                │
│                    │  (XGBoost)       │                                │
│                    └────────┬─────────┘                                │
│                              │                                          │
│                              ▼                                          │
│          ┌─────────────────────────────────────────┐                   │
│          │  Churn Probability: 78%  🔴 HIGH RISK   │                   │
│          │                                         │                   │
│          │  Top Risk Factors:                      │                   │
│          │  • 12 consecutive fails on level 156    │                   │
│          │  • Session frequency dropped 62%        │                   │
│          │  • No purchases in 30 days              │                   │
│          │                                         │                   │
│          │  Recommended Actions:                   │                   │
│          │  • Offer free booster pack              │                   │
│          │  • Reduce level 156 difficulty          │                   │
│          │  • Send re-engagement notification      │                   │
│          └─────────────────────────────────────────┘                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Feature Engineering for Churn

```python
from google.cloud import bigquery
import pandas as pd

def build_churn_features(lookback_days: int = 7) -> pd.DataFrame:
    """Build user features for churn prediction."""

    client = bigquery.Client()

    query = f"""
    WITH user_sessions AS (
        SELECT
            user_id,
            DATE(timestamp) as session_date,
            COUNT(DISTINCT session_id) as sessions,
            COUNTIF(event_type = 1002) as levels_completed,
            COUNTIF(event_type = 1003) as levels_failed,
            SUM(CAST(JSON_VALUE(parameters, '$.duration_sec') AS INT64)) as playtime_sec
        FROM `project.analytics.events`
        WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL {lookback_days + 14} DAY)
        GROUP BY user_id, DATE(timestamp)
    ),

    user_current AS (
        SELECT
            user_id,
            MAX(session_date) as last_session_date,
            SUM(sessions) as total_sessions,
            SUM(levels_completed) as total_levels_completed,
            SUM(levels_failed) as total_levels_failed,
            SUM(playtime_sec) / 3600.0 as total_playtime_hours
        FROM user_sessions
        WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL {lookback_days} DAY)
        GROUP BY user_id
    ),

    user_previous AS (
        SELECT
            user_id,
            SUM(sessions) as prev_sessions,
            SUM(levels_completed) as prev_levels_completed
        FROM user_sessions
        WHERE session_date >= DATE_SUB(CURRENT_DATE(), INTERVAL {lookback_days * 2} DAY)
          AND session_date < DATE_SUB(CURRENT_DATE(), INTERVAL {lookback_days} DAY)
        GROUP BY user_id
    ),

    user_progress AS (
        SELECT
            user_id,
            MAX(CAST(JSON_VALUE(parameters, '$.level_id') AS INT64)) as current_level,
            -- Consecutive fails on current level
            COUNTIF(
                event_type = 1003 AND
                CAST(JSON_VALUE(parameters, '$.level_id') AS INT64) = (
                    SELECT MAX(CAST(JSON_VALUE(parameters, '$.level_id') AS INT64))
                    FROM `project.analytics.events` e2
                    WHERE e2.user_id = events.user_id
                )
            ) as consecutive_fails_current_level
        FROM `project.analytics.events` events
        WHERE event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
          AND event_type IN (1001, 1002, 1003)
        GROUP BY user_id
    ),

    user_monetization AS (
        SELECT
            user_id,
            COUNT(*) as purchase_count_lifetime,
            SUM(CAST(JSON_VALUE(parameters, '$.price_usd') AS FLOAT64)) as revenue_lifetime,
            MAX(event_date) as last_purchase_date
        FROM `project.analytics.events`
        WHERE event_type = 2001  -- purchase_completed
        GROUP BY user_id
    ),

    user_install AS (
        SELECT
            user_id,
            MIN(event_date) as install_date
        FROM `project.analytics.events`
        WHERE event_type = 3001  -- app_install
        GROUP BY user_id
    )

    SELECT
        c.user_id,

        -- Engagement metrics (current period)
        COALESCE(c.total_sessions, 0) as sessions_7d,
        COALESCE(c.total_levels_completed, 0) as levels_completed_7d,
        COALESCE(c.total_levels_failed, 0) as levels_failed_7d,
        COALESCE(c.total_playtime_hours, 0) as playtime_hours_7d,

        -- Engagement trend (current vs previous period)
        SAFE_DIVIDE(c.total_sessions, NULLIF(p.prev_sessions, 0)) as session_trend,
        SAFE_DIVIDE(c.total_levels_completed, NULLIF(p.prev_levels_completed, 0)) as completion_trend,

        -- Recency
        DATE_DIFF(CURRENT_DATE(), c.last_session_date, DAY) as days_since_last_session,

        -- Progress metrics
        COALESCE(pr.current_level, 0) as current_level,
        COALESCE(pr.consecutive_fails_current_level, 0) as consecutive_fails,

        -- Monetization
        COALESCE(m.purchase_count_lifetime, 0) as purchases_lifetime,
        COALESCE(m.revenue_lifetime, 0) as revenue_lifetime,
        DATE_DIFF(CURRENT_DATE(), m.last_purchase_date, DAY) as days_since_purchase,

        -- Tenure
        DATE_DIFF(CURRENT_DATE(), i.install_date, DAY) as days_since_install,

        -- Target: Did user churn? (no activity in next 7 days)
        -- This would be filled retroactively for training data
        NULL as churned

    FROM user_current c
    LEFT JOIN user_previous p USING (user_id)
    LEFT JOIN user_progress pr USING (user_id)
    LEFT JOIN user_monetization m USING (user_id)
    LEFT JOIN user_install i USING (user_id)
    """

    return client.query(query).to_dataframe()
```

### Churn Model Training

```python
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, precision_recall_curve
import shap

class ChurnPredictor:
    """Predict probability of player churn."""

    def __init__(self):
        self.model = xgb.XGBClassifier(
            n_estimators=200,
            max_depth=5,
            learning_rate=0.1,
            subsample=0.8,
            colsample_bytree=0.8,
            scale_pos_weight=3,  # Handle class imbalance
            random_state=42,
            use_label_encoder=False,
            eval_metric='auc'
        )
        self.feature_columns = [
            'sessions_7d', 'levels_completed_7d', 'levels_failed_7d',
            'playtime_hours_7d', 'session_trend', 'completion_trend',
            'days_since_last_session', 'current_level', 'consecutive_fails',
            'purchases_lifetime', 'revenue_lifetime', 'days_since_purchase',
            'days_since_install'
        ]
        self.explainer = None

    def train(self, df: pd.DataFrame):
        """Train churn prediction model."""
        # Prepare data
        X = df[self.feature_columns].fillna(0)
        y = df['churned']

        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )

        # Train
        self.model.fit(
            X_train, y_train,
            eval_set=[(X_test, y_test)],
            early_stopping_rounds=20,
            verbose=False
        )

        # Evaluate
        y_pred_proba = self.model.predict_proba(X_test)[:, 1]
        auc = roc_auc_score(y_test, y_pred_proba)
        print(f"Churn Model AUC: {auc:.3f}")

        # Find optimal threshold
        precision, recall, thresholds = precision_recall_curve(y_test, y_pred_proba)
        f1_scores = 2 * (precision * recall) / (precision + recall + 1e-10)
        optimal_idx = f1_scores.argmax()
        self.threshold = thresholds[optimal_idx]
        print(f"Optimal threshold: {self.threshold:.3f}")

        # Setup SHAP for explanations
        self.explainer = shap.TreeExplainer(self.model)

    def predict(self, user_features: dict) -> dict:
        """Predict churn probability with explanations."""
        X = pd.DataFrame([user_features])[self.feature_columns].fillna(0)

        # Get probability
        proba = self.model.predict_proba(X)[0, 1]

        # Get SHAP values for explanation
        shap_values = self.explainer.shap_values(X)

        # Top risk factors
        feature_impacts = list(zip(self.feature_columns, shap_values[0]))
        feature_impacts.sort(key=lambda x: abs(x[1]), reverse=True)

        risk_factors = []
        for feature, impact in feature_impacts[:3]:
            if impact > 0:  # Contributing to churn
                risk_factors.append({
                    'feature': feature,
                    'value': user_features.get(feature, 0),
                    'impact': 'increases churn risk'
                })

        # Risk level
        if proba >= 0.7:
            risk_level = "HIGH"
        elif proba >= 0.4:
            risk_level = "MEDIUM"
        else:
            risk_level = "LOW"

        return {
            'churn_probability': round(proba, 3),
            'risk_level': risk_level,
            'risk_factors': risk_factors,
            'recommended_actions': self._get_recommendations(user_features, risk_factors)
        }

    def _get_recommendations(self, features: dict, risk_factors: list) -> list:
        """Generate personalized retention recommendations."""
        recommendations = []

        # Check consecutive fails
        if features.get('consecutive_fails', 0) >= 5:
            recommendations.append({
                'action': 'difficulty_adjustment',
                'message': f"Player stuck on level {features.get('current_level')} - consider offering hint or booster"
            })

        # Check engagement drop
        if features.get('session_trend', 1) < 0.5:
            recommendations.append({
                'action': 're_engagement',
                'message': "Significant drop in engagement - send personalized notification"
            })

        # Check monetization potential
        if features.get('purchases_lifetime', 0) > 0 and features.get('days_since_purchase', 999) > 14:
            recommendations.append({
                'action': 'special_offer',
                'message': "Previous buyer inactive - offer discounted bundle"
            })

        return recommendations
```

---

## Use Case 3: Automated Level Testing with RL Agents

### AI Agent Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RL AGENT FOR LEVEL TESTING                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                      Game Environment                             │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │  State:                                                     │  │  │
│  │  │  • Board configuration (9x9 grid)                          │  │  │
│  │  │  • Remaining moves                                         │  │  │
│  │  │  • Objective progress                                      │  │  │
│  │  │  • Available special candies                               │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              ▼                                    │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │  Actions: All valid swaps (typically 50-100 per state)     │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                    │  │
│  │                              ▼                                    │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │  Rewards:                                                   │  │  │
│  │  │  • +100 for completing level                               │  │  │
│  │  │  • +10 for each objective item cleared                     │  │  │
│  │  │  • +5 for creating special candies                         │  │  │
│  │  │  • -1 per move (encourage efficiency)                      │  │  │
│  │  │  • -50 for failing level                                   │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         RL Agent (PPO)                            │  │
│  │                                                                   │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │  │
│  │  │   CNN       │───▶│   Policy    │───▶│   Action    │          │  │
│  │  │   Encoder   │    │   Network   │    │   Sampler   │          │  │
│  │  └─────────────┘    └─────────────┘    └─────────────┘          │  │
│  │         │                                                        │  │
│  │         ▼                                                        │  │
│  │  ┌─────────────┐                                                 │  │
│  │  │   Value     │  (Estimates expected return)                    │  │
│  │  │   Network   │                                                 │  │
│  │  └─────────────┘                                                 │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### RL Environment Implementation

```python
import gymnasium as gym
import numpy as np
from typing import Tuple, Dict, Any, List

class Match3Environment(gym.Env):
    """Gym environment for match-3 puzzle games."""

    def __init__(self, level_config: dict):
        super().__init__()

        self.level_config = level_config
        self.grid_size = (9, 9)
        self.num_candy_types = 6

        # Observation space: board state + metadata
        self.observation_space = gym.spaces.Dict({
            'board': gym.spaces.Box(
                low=0, high=self.num_candy_types + 10,  # +10 for special types
                shape=(*self.grid_size, 1),
                dtype=np.int32
            ),
            'moves_remaining': gym.spaces.Box(low=0, high=100, shape=(1,), dtype=np.int32),
            'objective_progress': gym.spaces.Box(low=0, high=1, shape=(3,), dtype=np.float32),
        })

        # Action space: all possible swaps
        # Encoded as (row, col, direction) where direction: 0=right, 1=down
        self.action_space = gym.spaces.Discrete(self.grid_size[0] * self.grid_size[1] * 2)

        self.reset()

    def reset(self, seed=None, options=None) -> Tuple[Dict, Dict]:
        """Reset environment to initial state."""
        super().reset(seed=seed)

        self.board = self._generate_initial_board()
        self.moves_remaining = self.level_config.get('moves', 30)
        self.objectives = self._init_objectives()
        self.score = 0
        self.episode_moves = 0

        return self._get_observation(), {}

    def step(self, action: int) -> Tuple[Dict, float, bool, bool, Dict]:
        """Execute one step in the environment."""
        row, col, direction = self._decode_action(action)

        # Validate move
        if not self._is_valid_move(row, col, direction):
            return self._get_observation(), -1.0, False, False, {'invalid_move': True}

        # Execute swap
        reward = self._execute_swap(row, col, direction)

        # Process cascades
        cascade_reward = self._process_cascades()
        reward += cascade_reward

        # Update state
        self.moves_remaining -= 1
        self.episode_moves += 1

        # Check termination
        done = False
        truncated = False

        if self._objectives_complete():
            reward += 100  # Win bonus
            done = True
        elif self.moves_remaining <= 0:
            reward -= 50  # Lose penalty
            done = True
            truncated = True

        info = {
            'score': self.score,
            'moves_used': self.episode_moves,
            'objectives': self.objectives,
        }

        return self._get_observation(), reward, done, truncated, info

    def _execute_swap(self, row: int, col: int, direction: int) -> float:
        """Execute candy swap and return immediate reward."""
        # Get target position
        if direction == 0:  # Right
            target_row, target_col = row, col + 1
        else:  # Down
            target_row, target_col = row + 1, col

        # Swap candies
        self.board[row, col], self.board[target_row, target_col] = \
            self.board[target_row, target_col], self.board[row, col]

        # Find matches
        matches = self._find_matches()

        if not matches:
            # Invalid swap - swap back
            self.board[row, col], self.board[target_row, target_col] = \
                self.board[target_row, target_col], self.board[row, col]
            return -0.5

        # Process matches
        reward = self._process_matches(matches)
        return reward

    def _process_matches(self, matches: List[set]) -> float:
        """Clear matches and update objectives."""
        reward = 0

        for match in matches:
            match_size = len(match)

            # Base reward for match
            reward += match_size * 1.0

            # Bonus for larger matches
            if match_size >= 4:
                reward += 5  # Special candy created
            if match_size >= 5:
                reward += 10  # Super special candy

            # Update objectives
            for row, col in match:
                candy_type = self.board[row, col]
                self._update_objective(candy_type)

            # Clear matched candies
            for row, col in match:
                self.board[row, col] = 0

        return reward

    def _process_cascades(self) -> float:
        """Process falling candies and chain reactions."""
        total_reward = 0
        cascade_count = 0

        while True:
            # Apply gravity
            self._apply_gravity()

            # Fill empty spaces
            self._fill_empty_spaces()

            # Find new matches
            matches = self._find_matches()

            if not matches:
                break

            cascade_count += 1
            reward = self._process_matches(matches)
            total_reward += reward * (1 + cascade_count * 0.1)  # Cascade bonus

        return total_reward

    def _get_observation(self) -> Dict:
        """Get current observation."""
        return {
            'board': self.board.reshape(*self.grid_size, 1),
            'moves_remaining': np.array([self.moves_remaining], dtype=np.int32),
            'objective_progress': np.array([
                obj['current'] / obj['target']
                for obj in self.objectives.values()
            ][:3], dtype=np.float32),  # Pad to 3 objectives
        }

    def get_valid_actions(self) -> List[int]:
        """Return list of valid action indices."""
        valid = []
        for action in range(self.action_space.n):
            row, col, direction = self._decode_action(action)
            if self._is_valid_move(row, col, direction):
                valid.append(action)
        return valid

    # ... (helper methods: _find_matches, _apply_gravity, etc.)
```

### Training the Agent

```python
from stable_baselines3 import PPO
from stable_baselines3.common.vec_env import SubprocVecEnv
from stable_baselines3.common.callbacks import EvalCallback
import torch.nn as nn

class Match3PolicyNetwork(nn.Module):
    """Custom CNN policy for match-3 games."""

    def __init__(self, observation_space, action_space):
        super().__init__()

        # CNN for board state
        self.conv = nn.Sequential(
            nn.Conv2d(1, 32, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.Conv2d(32, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.Conv2d(64, 64, kernel_size=3, padding=1),
            nn.ReLU(),
            nn.Flatten(),
        )

        # Calculate conv output size
        conv_out_size = 64 * 9 * 9

        # Combine with metadata
        self.fc = nn.Sequential(
            nn.Linear(conv_out_size + 4, 256),  # +4 for moves and objectives
            nn.ReLU(),
            nn.Linear(256, 128),
            nn.ReLU(),
        )

        # Policy head
        self.policy = nn.Linear(128, action_space.n)

        # Value head
        self.value = nn.Linear(128, 1)

    def forward(self, obs):
        board = obs['board'].float()
        moves = obs['moves_remaining'].float()
        objectives = obs['objective_progress'].float()

        # Process board
        x = self.conv(board)

        # Combine with metadata
        metadata = torch.cat([moves, objectives], dim=-1)
        x = torch.cat([x, metadata], dim=-1)

        x = self.fc(x)

        return self.policy(x), self.value(x)


def train_level_agent(level_configs: List[dict], total_timesteps: int = 1_000_000):
    """Train RL agent on a set of levels."""

    # Create vectorized environments
    def make_env(level_config):
        def _init():
            return Match3Environment(level_config)
        return _init

    envs = SubprocVecEnv([make_env(cfg) for cfg in level_configs[:8]])

    # Create agent
    model = PPO(
        "MultiInputPolicy",
        envs,
        learning_rate=3e-4,
        n_steps=2048,
        batch_size=64,
        n_epochs=10,
        gamma=0.99,
        gae_lambda=0.95,
        clip_range=0.2,
        ent_coef=0.01,
        verbose=1,
        tensorboard_log="./tensorboard/match3_agent/"
    )

    # Evaluation callback
    eval_env = Match3Environment(level_configs[0])
    eval_callback = EvalCallback(
        eval_env,
        best_model_save_path="./models/",
        log_path="./logs/",
        eval_freq=10000,
        n_eval_episodes=100
    )

    # Train
    model.learn(
        total_timesteps=total_timesteps,
        callback=eval_callback,
        progress_bar=True
    )

    return model


def evaluate_level_with_agent(model, level_config: dict, n_episodes: int = 1000) -> dict:
    """Use trained agent to evaluate level difficulty."""

    env = Match3Environment(level_config)
    results = []

    for _ in range(n_episodes):
        obs, _ = env.reset()
        done = False
        episode_reward = 0

        while not done:
            action, _ = model.predict(obs, deterministic=False)
            obs, reward, done, truncated, info = env.step(action)
            episode_reward += reward
            done = done or truncated

        results.append({
            'completed': info.get('objectives_complete', False),
            'moves_used': info['moves_used'],
            'score': info['score'],
        })

    # Aggregate results
    completions = sum(1 for r in results if r['completed'])

    return {
        'ai_pass_rate': completions / n_episodes,
        'ai_avg_moves': np.mean([r['moves_used'] for r in results if r['completed']]) if completions > 0 else None,
        'ai_avg_score': np.mean([r['score'] for r in results]),
        'estimated_human_pass_rate': completions / n_episodes * 0.7,  # Humans typically 70% of AI
        'difficulty_assessment': 'Easy' if completions / n_episodes > 0.8 else 'Hard' if completions / n_episodes < 0.4 else 'Medium'
    }
```

---

## Putting It All Together: ML-Powered Level Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE ML-POWERED LEVEL PIPELINE                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. LEVEL DESIGN                                                        │
│  ┌──────────────┐                                                       │
│  │ Level Editor │  Designer creates new level                           │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  2. FEATURE EXTRACTION                                                  │
│  ┌──────────────┐                                                       │
│  │ Extract      │  Grid, objectives, blockers, etc.                     │
│  │ Features     │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  3. DIFFICULTY PREDICTION                                               │
│  ┌──────────────┐                                                       │
│  │ ML Model     │  Predict pass rate, moves, frustration               │
│  │ (XGBoost)    │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  4. AI SIMULATION                                                       │
│  ┌──────────────┐                                                       │
│  │ RL Agent     │  Play level 1000x to validate prediction             │
│  │ (PPO)        │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  5. REVIEW & ADJUST                                                     │
│  ┌──────────────┐                                                       │
│  │ Dashboard    │  Show predictions vs simulations                      │
│  │              │  Recommend adjustments                                │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  6. A/B TESTING (Optional)                                              │
│  ┌──────────────┐                                                       │
│  │ Release to   │  Test with small player segment                       │
│  │ 5% of users  │                                                       │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  7. FULL RELEASE                                                        │
│  ┌──────────────┐                                                       │
│  │ Monitor &    │  Track actual vs predicted metrics                    │
│  │ Learn        │  Retrain models with new data                         │
│  └──────────────┘                                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Integration API

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import List, Dict, Any

app = FastAPI(title="Level Analytics API")

# Load models
difficulty_predictor = DifficultyPredictor.load("models/difficulty_model.pkl")
churn_predictor = ChurnPredictor.load("models/churn_model.pkl")
rl_agent = PPO.load("models/rl_agent.zip")


class LevelConfig(BaseModel):
    grid: List[List[str]]
    objectives: List[Dict[str, Any]]
    moves: int
    blockers: List[Dict[str, Any]] = []
    special_elements: Dict[str, Any] = {}


class UserFeatures(BaseModel):
    user_id: str
    sessions_7d: int
    levels_completed_7d: int
    current_level: int
    consecutive_fails: int
    purchases_lifetime: int
    days_since_install: int


@app.post("/api/v1/level/predict-difficulty")
async def predict_difficulty(level: LevelConfig):
    """Predict difficulty metrics for a level configuration."""
    try:
        # Extract features
        features = extract_features(level.dict())

        # Get ML prediction
        ml_prediction = difficulty_predictor.predict(features)

        # Run AI simulation
        ai_results = evaluate_level_with_agent(
            rl_agent, level.dict(), n_episodes=100
        )

        return {
            "ml_prediction": ml_prediction,
            "ai_simulation": ai_results,
            "combined_assessment": {
                "recommended_pass_rate": (
                    ml_prediction['predicted_pass_rate'] * 0.6 +
                    ai_results['estimated_human_pass_rate'] * 0.4
                ),
                "confidence": "high" if abs(
                    ml_prediction['predicted_pass_rate'] -
                    ai_results['estimated_human_pass_rate']
                ) < 0.1 else "medium"
            }
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.post("/api/v1/user/predict-churn")
async def predict_churn(user: UserFeatures):
    """Predict churn probability for a user."""
    try:
        prediction = churn_predictor.predict(user.dict())
        return prediction
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@app.get("/api/v1/levels/at-risk")
async def get_at_risk_levels(min_attempts: int = 1000, max_pass_rate: float = 0.3):
    """Get levels where players are struggling."""
    client = bigquery.Client()

    query = f"""
    SELECT
        level_id,
        pass_rate,
        avg_moves,
        retry_rate,
        unique_players
    FROM `project.analytics.level_metrics`
    WHERE date = CURRENT_DATE() - 1
      AND attempts >= {min_attempts}
      AND pass_rate <= {max_pass_rate}
    ORDER BY unique_players DESC
    LIMIT 20
    """

    results = client.query(query).to_dataframe()
    return results.to_dict(orient='records')
```

---

## Summary

| Use Case | Model Type | Key Features | Business Impact |
|----------|------------|--------------|-----------------|
| Difficulty Prediction | Gradient Boosting | Grid structure, objectives, blockers | Reduce bad level releases |
| Churn Prediction | XGBoost | Engagement trends, stuck levels, monetization | Proactive retention |
| Level Testing | RL (PPO) | Board state, valid actions, rewards | Faster QA, better balance |

**Key Takeaways:**

1. **Feature engineering is crucial** - Domain knowledge about game mechanics creates the best predictors
2. **Combine ML and simulation** - Statistical models + AI agents provide robust estimates
3. **Close the feedback loop** - Continuously retrain models with new player data
4. **Explainability matters** - Use SHAP values to understand why predictions are made
5. **Start simple** - Gradient boosting often beats deep learning for tabular game data

---

## Further Reading

- [Reinforcement Learning for Game Testing](https://arxiv.org/abs/2103.13798)
- [Player Modeling in Games](https://ieeexplore.ieee.org/document/6932866)
- [A/B Testing Best Practices](https://www.exp-platform.com/Documents/2014%20experimentersRulesOfThumb.pdf)

---

*Part of the [Game Analytics Series](./index#game-analytics)*
