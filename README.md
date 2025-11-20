# Gameplay Telemetry Collection for SuperTuxKart

The goal of this exercise is to extract gameplay telemetry from SuperTuxKart, compute data-driven frustration indicators, and analyze which tracks tend to produce higher frustration. The analysis uses only racing sessions played manually under different tracks and difficulty settings. This report describes how the gameplay telemetry was collected, how the frustration score was constructed, and the insights obtained from track-based analysis

---

### 1. Installation and Setup

The game was installed and built on Windows OS following the official instructions:

`https://github.com/NimmiW/stk-code/blob/master/INSTALL.md`

The repository was cloned, configured using CMake, and built successfully. The resulting executable was verified to run correctly before code modifications.

---

### 2. Codebase Modifications for Data Logging

Data logging functionality was added to `src/karts/controller/player_controller.hpp` and `src/karts/controller/player_controller.cpp`.

#### 2.1 Added Members and Method (player_controller.hpp)

```
    std::ofstream m_log_file;
    uint64_t m_game_id;
    void logData();
```

#### 2.2 Initialization in Constructor (player_controller.cpp)

A unique game session ID is generated, and the CSV log file (player_log.csv) is initialized in append mode. The header row is inserted only if the file is empty.

```
    std::random_device rd;
    std::mt19937_64 gen(rd());
    std::uniform_int_distribution<uint64_t> dis(100000, 999999);

    m_game_id = dis(gen);

    m_log_file.open("player_log.csv", std::ios::app);
    if (m_log_file.is_open())
    {
        m_log_file.seekp(0, std::ios::end);
        if (m_log_file.tellp() == 0)
        {
            m_log_file << "game_id,time,track,difficulty,kart_type,steer,accel,speed,brake,on_ground,x,y,z,energy\n";
        }
    }
```

#### 2.3 Logging Method Implementation

The logData() method extracts telemetry information such as steering input, acceleration, speed, braking, on-ground status, kart position, and energy. Metadata such as track name, difficulty level, and kart type are also recorded.

```
    void PlayerController::logData()
    {
        if (!m_log_file.is_open()) return;

            using namespace std::chrono;

            uint64_t ms = duration_cast<milliseconds>(
                            steady_clock::now().time_since_epoch()).count();

            std::string track      = RaceManager::get()->getTrackName();
            std::string kart_type  = m_kart->getKartProperties()->getNonTranslatedName();
            std::string difficulty = RaceManager::get()->getDifficultyAsString(RaceManager::get()->getDifficulty());

            float steer = m_controls->getSteer();
            float accel = m_controls->getAccel();
            float speed = m_kart->getSpeed();
            bool  brake = m_controls->getBrake();
            bool  on_ground = m_kart->isOnGround();

            auto pos = m_kart->getXYZ();
            float energy = m_kart->getEnergy();

            m_log_file
                << m_game_id << ","
                << ms << ","
                << track << ","
                << difficulty << ","
                << kart_type << ","
                << steer << ","
                << accel << ","
                << speed << ","
                << (brake ? 1 : 0) << ","
                << (on_ground ? 1 : 0) << ","
                << pos[0] << "," << pos[1] << "," << pos[2] << ","
                << energy << ","
                << "\n";
    }
```

#### 2.4 Integrating Logging into Game Loop

`logData()` was called inside the `PlayerController::update()` function to capture telemetry at each frame update.

---

### 3. Gameplay Data Collection Procedure

All tracks were unlocked manually by modifying the Windows AppData STK configuration files. A single user played all sessions to maintain consistency in play style. Telemetry for each session is stored in a csv file. A unique `game_id` was assigned to every race session, ensuring that all frames from the same race could be grouped.

i. Number of tracks: 21
ii. Difficulty levels: Novice, Intermediate, Expert, SuperTux (4 total)
iii. Attempts per track per difficulty: 2
iv. Total gameplay sessions:

21 tracks × 4 difficulties × 2 runs = 168 total games

-> `player_log_round_1.csv` contains the telemetry data for 1st run

-> `player_log_round_2.csv` contains the telemetry data for 2nd run.

---

### 4. Data Fields Logged

Each log entry contains the following:

```
game_id: Unique session identifier
time: Timestamp (ms)
track: Track name
difficulty: Difficulty level
kart_type: Selected kart
steer: Steering input
accel: Acceleration input
speed: Current speed
brake: Braking input
on_ground: Whether the kart is grounded
x, y, z: Position coordinates
energy: Kart energy level
```

---

### 5. Computing Frustration Indicators

For each game session, three behavioral frustration signals were extracted:

#### 5.1 Off-Ground Ratio (%)

offGroundRatio = $\frac{frames\ where\ on\_ground = 0}{total\ frames} \times 100$

A high value means the kart is frequently airborne or off-track events that disrupt flow and cause frustration.

#### 5.2 Sudden Speed Drops

Using the frame-to-frame differences:

```
drops = np.diff(speed)
speed_drop_count = np.sum(drops < -5)
```

Large speed loss typically corresponds to collisions, falling off-track, or hitting obstacles.

#### 5.3 Steering Instability

`steer_changes = np.sum(np.diff(np.sign(steer)) != 0)`

A sign flip indicates a switch from left to right steering (or vice versa).
Frequent flips → sharp turns, overcorrections, or difficulty keeping the kart stable.

---

### 6. Frustration Score Construction

To create a single frustration metric per session, a weighted score was defined:

`FrustrationScore = 0.4 X OffGroundRatio + 0.4 X SpeedDropCount + 0.2 X SteerChanges`

Rationale for weights:

- 0.4 for off-ground events: breaking flow and control is highly frustrating
- 0.4 for sudden speed loss: often punishes players severely
- 0.2 for steering instability: contributes to difficulty but less severe

The final scores were saved:

`result_df.to_csv('frustration_scores.csv', index=False)`

---

### 7. Analysis

#### 7.1 Distribution of Frustration Scores

![Distribution of Frustration Score](/plots/distribution_of_frustration_scores.png "Distribution of Frustration Score")

Observations:

- Most races fall within moderate frustration, but a small number of sessions show very high scores.
- These high-frustration cases often correspond to tracks with obstacles, jumps, or tight turns.

This suggests the frustration metric successfully captures variability across tracks.

#### 5.2 Average Frustration by Track

![Average Frustration by Track](/plots/average_frustration_by_tracks.png "Average Frustration by Track")

Findings:

- Tracks like volcano_island, black_forest, and xr591 repeatedly produced high frustration, due to complex geometry and elevation changes.
- Tracks such as lighthouse or minigolf showed consistently low scores, indicating smoother layouts.

This proves that track design directly influences player frustration, independent of difficulty.

---

### 8. Conclusion

Through custom telemetry logging and analysis, I computed a frustration metric for each SuperTuxKart race session. By focusing on track-level analysis and frustration distribution, the results reveal clear differences in how track design affects player experience. Tracks with more jumps, sharp turns, or obstacles produce higher frustration, while smoother tracks show lower scores.

These findings can guide developers by identifying:

- tracks that require redesign or balancing
- track sections causing excessive airborne time or speed loss
- steering-related difficulty patterns

Overall, this data-driven methodology provides valuable insights for improving track quality, fairness, and gameplay enjoyment.

---

### 9. Limitations

- Single-player bias: Only one player participated; gameplay skill may influence results.
- Limited samples: Each track–difficulty combination was played only twice, which may not fully represent performance variability.
- Controlled environment: Real-world variation (e.g., different player skills) is not captured.

```

```
