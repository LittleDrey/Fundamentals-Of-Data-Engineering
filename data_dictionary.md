## 📚 Dicionário de Dados (Camada Silver)

Este dicionário documenta a estrutura, a tipagem e a semântica das tabelas disponíveis na **Camada Silver** do Data Lake. Estas tabelas já passaram por processos de *Data Cleaning*, *Flattening* (desaninhamento de JSONs), padronização de nomenclatura (*snake_case*) e tipagem forte.

---

### 1. Tabela: `estadios` (Venues)
Dimensão contendo os metadados dos locais onde as partidas ocorrem. Os dados foram normalizados (1ª Forma Normal) para separar a cidade e o estado.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `stadium_sk` | Integer | Chave substituta (*Surrogate Key*) gerada internamente para garantir unicidade do estádio. |
| `stadium_id` | Integer | Identificador único do estádio na fonte (API). |
| `stadium_name` | String | Nome oficial do estádio. |
| `stadium_city` | String | Cidade onde o estádio fica localizado. |
| `stadium_state` | String | Estado da localização do estádio. |
| `stadium_road` | String | Rua em que o estádio foi construído. |
| `stadium_district` | String | Bairro onde se o encontra. |
| `stadium_country` | String | País de origem. |
| `stadium_capacity` | Integer | Capacidade máxima de público. |
| `stadium_surface` | String | Tipo de gramado (grass, artificial, etc.). |
| `source_file` | String | Caminho do arquivo de origem na Raw/Bronze (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |

---

### 2. Tabela: `eventos` (Events)
Tabela transacional (Fato) de altíssima granularidade. Registra cada ocorrência (gols, cartões, substituições) minuto a minuto dentro das partidas.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `event_sk` | Integer | Chave substituta (*Surrogate Key*) gerada internamente para garantir unicidade do evento. |
| `match_src_id` | Integer | Chave estrangeira (FK) conectando à partida onde o evento ocorreu. |
| `minute` | Integer | Minuto normal do jogo em que o evento ocorreu (valor absoluto, corrigido). |
| `extra_time` | Integer | Minutos de acréscimo no momento do evento (nulo/0 se não houver). |
| `match_minute_abs`| Integer | Minuto absoluto do evento (soma do minuto + extra time) para ordenação cronológica. |
| `team_id` | Integer | ID do time que realizou o evento. |
| `team_name` | String | Nome do time que realizou o evento. |
| `player_id` | Integer | ID do jogador principal do evento (quem fez o gol, tomou cartão). |
| `player_name` | String | Nome do jogador principal do evento. |
| `assist_player_id`| Integer | ID do jogador que deu a assistência (nulo se não aplicável). |
| `assist_player_name`| String | Nome do jogador que deu a assistência. |
| `event_type` | String | Categoria do evento (Goal, Card, Subst). |
| `event_detail` | String | Detalhe específico (Normal Goal, Yellow Card, etc.). |
| `comments` | String | Comentários adicionais da arbitragem ou sistema. |
| `source_file` | String | Caminho do arquivo de origem (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |

---

### 3. Tabela: `jogadores` (Players)
Dimensão contendo o perfil biológico e demográfico dos atletas.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `player_sk` | Integer | Chave substituta do jogador no Data Lake (*Surrogate Key*). |
| `player_id` | Integer | Identificador do jogador na fonte (API). |
| `player_age` | Integer | Idade do jogador. |
| `player_firstname` | String | Primeiro nome. |
| `player_lastname` | String | Sobrenome. |
| `player_name` | String | Nome conhecido do jogador (ex: "Nino"). |
| `is_injured` | Boolean | Coluna que evidencia se o jogador está lesionado (True/False). |
| `player_nationality` | String | Nacionalidade do jogador. |
| `player_birth_date` | Date | Data de nascimento. |
| `player_country` | String | País de nascimento. |
| `player_place` | String | Cidade/Estado de nascimento. |
| `player_height_cm` | Integer | Altura do jogador em centímetros (sanitizado via regex). |
| `player_weight_kg` | Integer | Peso do jogador em quilogramas (sanitizado via regex). |
| `source_file` | String | Caminho do arquivo de origem (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |

---

### 4. Tabela: `partidas` (Matches)
Tabela principal consolidando o resultado final e o status de cada jogo do campeonato.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `match_sk` | Integer | Chave substituta do jogador no Data Lake (*Surrogate Key*). |
| `match_id` | Integer | Identificador único da partida na fonte. |
| `torunament_id` | Integer | ID da liga/torneio correspondente. |
| `venue_id` | Integer | ID do estádio onde ocorreu o jogo. |
| `match_date` | Timestamp | Data e hora oficial de início da partida. |
| `match_season_year` | Integer | Ano da temporada correspondente ao jogo. |
| `match_round` | String | Rodada do campeonato (ex: "Regular Season - 1"). |
| `match_status` | String | Status final (Match Finished, Cancelled, etc.). |
| `match_referee` | String | Nome do árbitro principal. |
| `home_team_id` | Integer | ID do time mandante. |
| `away_team_id` | Integer | ID do time visitante. |
| `home_team_goals` | Integer | Gols totais do time mandante (Placar Final). |
| `away_team_goals` | Integer | Gols totais do time visitante (Placar Final). |
| `home_team_halftime_goals`| Integer | Gols do time mandante apenas no 1º tempo. |
| `away_team_halftime_goals`| Integer | Gols do time visitante apenas no 1º tempo. |
| `home_team_fulltime_goals`| Integer | Gols do time mandante no tempo regulamentar (90 min). |
| `away_team_fulltime_goals`| Integer | Gols do time visitante no tempo regulamentar (90 min). |
| `is_finished` | Boolean | Flag de negócio indicando se a partida foi totalmente encerrada. |
| `source_file` | String | Caminho do arquivo de origem (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |

---

### 5. Tabela: `times` (Teams)
Dimensão contendo os dados dos clubes de futebol.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `team_sk` | Integer | Chave substituta interna do Data Lake para o time. |
| `team_id` | Integer | ID do time na fonte original. |
| `team_name` | String | Nome oficial do clube (ex: "Flamengo"). |
| `team_code` | String | Sigla do time (ex: "FLA"). Preenchido com "N/A" se nulo. |
| `country_name` | String | País de origem do clube. |
| `founded_year` | Integer | Ano de fundação do clube. |
| `is_national` | Boolean | Flag indicando se é um time local/nacional (Regra: True para 'Brazil'). |
| `source_file` | String | Caminho do arquivo de origem (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |

---

### 6. Tabela: `torneios` (Leagues)
Dimensão que cataloga as edições dos campeonatos.

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `tournament_sk` | Integer | Chave substituta interna do Data Lake para torneios. |
| `tornament_id` | Integer | ID do torneio na fonte. |
| `tournament_name` | String | Nome da liga/copa (ex: "Serie A"). |
| `country_name` | String | País sede do torneio. |
| `season_year` | Integer | Ano de realização da edição. |
| `season_start` | Date | Data de abertura do torneio. |
| `season_end` | Date | Data de encerramento do torneio. |
| `is_current_season`| Boolean | Flag indicando se é a temporada que está ocorrendo atualmente. |
| `source_file` | String | Caminho do arquivo de origem (Linhagem de Dados). |
| `ingestion_date` | Timestamp | Data e hora exata da ingestão no fuso America/Sao_Paulo. |
