# Riot Games API - Teamfight Tactics Data Project

## Project Overview
This project use the **Riot Games API** to collect and analyze data from **Teamfight Tactics (TFT)** matches. It pulls real-time data for top-ranked players, processes the match information, and stores it in a PostgreSQL database for further analysis.

The project includes:
- Fetching **PUUIDs** and **Summoner names** for the top players in the **Challenger**, **Grandmaster**, and **Master** leagues.
- Retrieving match data for these players, including **game stats** and **traits** used in matches.
- Storing the processed data in a **PostgreSQL database**.

## Features
- **Leaderboard Scraping:** Fetches data for the top 50 players across multiple tiers (Challenger, Grandmaster, and Master).
- **Match Details:** Retrieves detailed match information including **summoner placement**, **game stats**, and **traits** used during gameplay.
- **Database Integration:** Uses **SQLAlchemy** and **Pangres** to upsert data into a PostgreSQL database, allowing for efficient storage and retrieval.

## Getting Started

### Prerequisites

Make sure you have the following installed:
- **Python 3.12+**
- **PostgreSQL**
- **Pip** for managing Python packages
- **Riot API Key** for accessing the Riot Games data.

You can obtain a Riot API Key from [Riot Developer Portal](https://developer.riotgames.com/).

### Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/yourusername/repository-name.git
   cd repository-name

2. **Create and activate a virtual environment:**
    ```bash
    python -m venv venv
    venv\Scripts\activate   # On Windows
    source venv/bin/activate  # On macOS/Linux

3. **Install the required dependencies:
    ```bash
    pip install -r  requirements.txt

4. **Set up environment variables:**
   - **Create a `.env` file in the root of the project and add your API key and database credentials:**
   ```bash
   riot_api=your_riot_api_key
   db_username=your_db_username
   db_password=your_db_password
   db_host=localhost
   db_port=5432
   db_name=your_db_name

### Running the Project

To run the project and fetch data:
1. Execute the script:
   ```bash
   python your_script_name.py
2. The data will be fetched and inserted into your PostgreSQL database.

## Code Overview

1. Environment Setup
The script uses a `.env` file to securely store API keys and database credentials. The `python-dotenv` library loads the environment variables, ensuring the API key and database credentials are kept private.
    ```python
    from dotenv import load_dotenv
    load_dotenv()
    ```
    
2. Fetching Player Information
Functions are defined to retrieve player data from the Riot Games API:
- `get_puuid_by_Id(summonerId)`: Fetches the **PUUID** (Player Unique ID) for a given summoner ID.
- `get_name_and_tag(puuid)`: Fetches the game name and tag line of the player using the **PUUID**.

  ```python
  def get_puuid_by_Id(summonerId=None):
    root_url = 'https://na1.api.riotgames.com/tft/summoner/v1/summoners/'
    link = f'{root_url}{summonerId}?api_key={api_key}'
    response = requests.get(link)
    return response.json()['puuid']
  ```
  ```python
  def get_name_and_tag(puuid=None):
    link = f'https://americas.api.riotgames.com/riot/account/v1/accounts/by-puuid/{puuid}?api_key={api_key}'
    response = requests.get(link)

    id = {
        'gameName' : response.json()['gameName'],
        'tagLine' : response.json()['tagLine']
    }
    return id['gameName']
  ```
3. Retrieving Ladder Data
The `get_ladder_ids()` function pulls data from the top players in the **Challenger, Grandmaster, and Master** leagues.
    ```python
    def get_ladder_ids():
      root = 'https://na1.api.riotgames.com/'
      chall = 'tft/league/v1/challenger?queue=RANKED_TFT'
      gm = 'tft/league/v1/grandmaster?queue=RANKED_TFT'
      masters = 'tft/league/v1/master?queue=RANKED_TFT'
  
      chall_response = requests.get(root + chall + '&api_key=' + api_key)
      gm_response = requests.get(root + gm + '&api_key=' + api_key)
      masters_response = requests.get(root + masters + '&api_key=' + api_key)
  
      chall_df = pd.DataFrame(chall_response.json()['entries']).sort_values('leaguePoints', ascending=False).reset_index(drop=True)
      gm_df = pd.DataFrame(gm_response.json()['entries']).sort_values('leaguePoints', ascending=False).reset_index(drop=True)
      masters_df = pd.DataFrame(masters_response.json()['entries']).sort_values('leaguePoints', ascending=False).reset_index(drop=True)
  
      ladder = pd.concat([chall_df, gm_df, masters_df]).reset_index(drop=True)[:50]
      ladder = ladder.reset_index().drop(columns=['rank']).rename(columns={'index':'rank'})
      ladder['rank']+=1
      return ladder['summonerId']
    ```
4. Fetching Match Data
   - The function `get_match()` retrieves match IDs for a player, starting from a specified point and limiting the number of matches fetched.
   - The function `get_match_data()` retrieves detailed match data for each match ID.
    ```python
    def get_match(puuid=None, start=0, count=20):
      root = 'https://americas.api.riotgames.com/'
      endpoint = f'/tft/match/v1/matches/by-puuid/{puuid}/ids'
      query_params = f'?start={start}&count={count}'
  
      response = requests.get(root + endpoint + query_params + '&api_key=' + api_key)
  
      return response.json()

    def get_match_data(match_id=None):
      root = 'https://americas.api.riotgames.com/'
      endpoint = f'/tft/match/v1/matches/{match_id}'
  
      response = requests.get(root + endpoint + '?api_key=' + api_key)
      return response.json()

5. Processing Match Data
The function `process_matches()` processes the match data by extracting details about the trais and other statistics, such as player placement, damage, and units used during the match.
    ```python
    def process_matches(matchjson, puuid):
    arcana = 0
    chrono = 0
    dragon = 0
    # Additional trait counters

    metadata = matchjson['metadata']
    info = matchjson['info']

    match_id = metadata['match_id']
    participants = metadata['participants']

    name_list = []
    for i in range(len(participants)):
        name = get_name_and_tag(puuid=participants[i])
        name_list.append(name)

    players = info['participants']
    game_creation = info['game_datetime']
    game_length = info['game_length']
    game_version = info['game_version']
    tft_set_number = info['tft_set_number']

    player = players[participants.index(puuid)]

    # Extract and process player statistics
    # Append the processed data to a DataFrame
    matchDF = pd.DataFrame({
        'match_id' : [match_id],
        'participants' : [name_list],
        'game_creation' : [game_creation],
        'game_length' : [game_length],
        # Other fields
    })

    return matchDF
    ```
6. Upserting Data into Database:
 The script connects to a PostgreSQL database and inserts or updates match data using the `upsert()` function from the `pangres` library.
    ```python
    from pangres import upsert
    from sqlalchemy import text, create_engine
    
    # Database connection setup
    connection_url = create_db_connection_string(db_username, db_password, db_host, db_port, db_name)
    
    db_engine = create_engine(connection_url, pool_recycle=3600)
    
    connection = db_engine.connect()
    
    upsert(con=connection, df=df, schema='soloq', table_name='regional_player_matches', create_table=True, create_schema=True, if_row_exists='update')
    
    connection.commit()
    ```
This script automates the extraction of high-level data (rankings, players, and matches) from the Riot Games API, processes the data, and then inserts it into a PostgreSQL database for further analysis.
