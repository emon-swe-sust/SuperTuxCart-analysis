# Gameplay Telemetry Collection for SuperTuxKart

This project extends the SuperTuxKart (STK) codebase to collect fine-grained gameplay telemetry data for analyzing frustration/difficulties levels across different tracks and difficulty settings. The modifications allow automatic logging of player inputs, kart state, track metadata, and positional information during gameplay.

---

### 1. Installation and Setup

The game was installed and built on Windows OS following the official instructions:

`https://github.com/NimmiW/stk-code/blob/master/INSTALL.md`

The repository was cloned, configured using CMake, and built successfully. The resulting executable was verified to run correctly before code modifications.

---

### 2. Codebase Modifications for Data Logging

Data logging functionality was added to src/karts/controller/player_controller.hpp and player_controller.cpp.

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

All tracks were unlocked manually by modifying the Windows AppData STK configuration files. A single user played all sessions to maintain consistency in play style. Telemetry for each session is stored in player_log.csv.

i. Number of tracks: 21
ii. Difficulty levels: Novice, Intermediate, Expert, SuperTux (4 total)
iii. Attempts per track per difficulty: 2
iv. Total gameplay sessions:

21 tracks × 4 difficulties × 2 runs = 168 total games

-> player_log_round_1.csv contains the telemetry data for 1st run.
-> player_log_round_2.csv contains the telemetry data for 2nd run.

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

### 5. Limitations

i. Single-player bias: Only one player participated; gameplay skill may influence results.
ii• Limited samples: Each track–difficulty combination was played only twice, which may not fully represent performance variability.
iii• Controlled environment: Real-world variation (e.g., different player skills) is not captured.
