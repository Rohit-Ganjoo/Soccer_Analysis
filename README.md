### Soccer Analysis
#### Head to Head Analysis of Teams:
```sql
-- Winner and Loser Count of Matches:

-- Subquery to get home team details
WITH HOME AS (
    SELECT 
        t.team_long_name, 
        m.away_team_api_id, 
        m.home_team_goal, 
        m.away_team_goal
    FROM 
        matches AS m
    JOIN 
        team AS t ON t.team_api_id = m.home_team_api_id
),

-- Subquery to get both home and away team details
HOME_AWAY AS (
    SELECT 
        h.team_long_name AS `Home Team`, 
        t.team_long_name AS `Away Team`, 
        h.home_team_goal, 
        h.away_team_goal
    FROM 
        home AS h
    JOIN 
        team t ON t.team_api_id = h.away_team_api_id
),

-- Subquery to determine the result of the match (Win, Lose, or Draw)
TEAM_GOAL_CTE AS (
    SELECT 
        `Home Team`, 
        `Away Team`, 
        home_team_goal, 
        away_team_goal,
        CASE 
            WHEN home_team_goal > away_team_goal THEN `Home Team`
            WHEN home_team_goal < away_team_goal THEN `Away Team`
            ELSE "Draw" 
        END AS Result
    FROM 
        home_away
),

-- Subquery to identify the winners and losers of each match
WIN_LOSE_CTE AS (
    SELECT 
        CASE 
            WHEN `Home Team` = Result THEN `Home Team`
            WHEN `Away Team` = Result THEN `Away Team` 
        END AS Winner,
        CASE 
            WHEN `Home Team` <> Result AND Result <> 'Draw' THEN `Home Team`
            WHEN `Away Team` <> Result AND Result <> 'Draw' THEN `Away Team` 
        END AS Loser,
        CASE 
            WHEN Result = 'Draw' THEN 'Draw' 
        END AS Remark
    FROM 
        team_goal_cte
)

-- Main query to count the number of matches won and lost by each team
SELECT 
    Winner, 
    Loser, 
    COUNT(*) AS Matches 
FROM 
    win_lose_cte
GROUP BY 
    1,2
HAVING 
    Winner IS NOT NULL
ORDER BY 
    Matches DESC;

```

#### What is the performance of each player over the years?
```sql
-- Performance Pivot:
WITH performance AS (
    SELECT 
        YEAR(pa.date) AS Year,
        p.player_name AS `Player Name`,
        ROUND(AVG(pa.overall_rating) OVER(PARTITION BY p.player_name, YEAR(pa.date) ORDER BY p.player_name, YEAR(pa.date)), 1) AS Performance_Rating 
    FROM 
        player AS p
        JOIN player_attributes AS pa ON pa.player_api_id = p.player_api_id
),
result_cte AS (
    SELECT 
        Year, 
        `Player Name`, 
        ROUND(AVG(Performance_Rating), 1) AS Performance 
    FROM 
        performance
    GROUP BY 
        2, 1
)
SELECT 
    `Player Name`, 
    SUM(CASE WHEN Year = 2007 THEN Performance ELSE 0 END) AS `2007`,
    SUM(CASE WHEN Year = 2008 THEN Performance ELSE 0 END) AS `2008`,
    SUM(CASE WHEN Year = 2009 THEN Performance ELSE 0 END) AS `2009`,
    SUM(CASE WHEN Year = 2010 THEN Performance ELSE 0 END) AS `2010`,
    SUM(CASE WHEN Year = 2011 THEN Performance ELSE 0 END) AS `2011`,
    SUM(CASE WHEN Year = 2012 THEN Performance ELSE 0 END) AS `2012`,
    SUM(CASE WHEN Year = 2013 THEN Performance ELSE 0 END) AS `2013`,
    SUM(CASE WHEN Year = 2014 THEN Performance ELSE 0 END) AS `2014`,
    SUM(CASE WHEN Year = 2015 THEN Performance ELSE 0 END) AS `2015`,
    SUM(CASE WHEN Year = 2016 THEN Performance ELSE 0 END) AS `2016`,
    SUM(CASE WHEN Year = 2017 THEN Performance ELSE 0 END) AS `2017`
FROM 
    result_cte
GROUP BY 
	1;




```

#### Players' performance based on their preferred foot:
```sql

DELIMITER //

CREATE PROCEDURE player_rating (IN foot_type VARCHAR(10))
BEGIN
    SELECT 
        p.player_name, 
        ROUND(AVG(pa.overall_rating), 1) AS Overall_Rating
    FROM 
        player AS p
        JOIN player_attributes AS pa ON pa.player_api_id = p.player_api_id
    WHERE 
        pa.preferred_foot = foot_type
    GROUP BY 
        p.player_name
    ORDER BY 
        Overall_Rating DESC
	LIMIT 5;
END //

DELIMITER ;

CALL player_rating('left');
```


#### Calculate the team's winning percentage and order it by the decreasing order of their winning percentage:
```sql
WITH Full_team AS (
    SELECT 
        home_team_api_id AS Team, 
        CASE 
            WHEN home_team_goal > away_team_goal THEN 1 
            ELSE 0 
        END AS flag
    FROM 
        matches
    UNION ALL 
    SELECT 
        away_team_api_id AS Team,
        CASE 
            WHEN home_team_goal < away_team_goal THEN 1 
            ELSE 0 
        END AS flag
    FROM 
        matches
),
Matches_list AS (
    SELECT 
        t.team_long_name AS Team, 
        SUM(ft.flag) AS Matches_Won, 
        COUNT(ft.flag) AS Matches_Played
    FROM 
        Full_team ft
    JOIN 
        team t ON t.team_api_id = ft.Team
    GROUP BY 
        1
)
SELECT 
    *, 
    ROUND((Matches_Won / Matches_Played) * 100, 2) AS Winning_Percentage
FROM 
    Matches_list
ORDER BY 
    Winning_Percentage DESC;

```

#### Age Group Performance
```sql
WITH Player_age AS (
    SELECT 
        (YEAR(date) - YEAR(birthday)) AS Age, 
        pa.overall_rating AS Overall_Rating,
        pa.potential AS Potential,
        pa.dribbling AS Dribbling,
        pa.curve AS Curve,
        pa.free_kick_accuracy AS Free_Kick_Accuracy,
        pa.long_passing AS Long_Pass,
        pa.ball_control AS Ball_Control,
        pa.acceleration AS Acceleration,
        pa.sprint_speed AS Sprint_Speed,
        pa.agility AS Agility,
        pa.reactions AS Reactions,
        pa.balance AS Balance,
        pa.shot_power AS Shot_Power,
        pa.jumping AS Jumping,
        pa.stamina AS Stamina,
        pa.strength AS Strength,
        pa.long_shots AS Long_Shot
    FROM 
        player AS p
    JOIN 
        player_attributes AS pa ON pa.player_api_id = p.player_api_id
)
SELECT 
    CASE 
        WHEN Age < 20 THEN 'Under 20' 
        WHEN Age >= 20 AND Age <= 25 THEN '20 to 25'
        WHEN Age > 25 AND Age <= 30 THEN '26 to 30'
        WHEN Age > 30 AND Age <= 35 THEN '31 to 35'
        WHEN Age > 35 AND Age <= 40 THEN '36 to 40'
        WHEN Age > 40 AND Age <= 45 THEN '41 to 45'
        ELSE '45 and Over' 
    END AS `Age Group`, 
    AVG(Overall_Rating) AS Average_Rating,
    AVG(Potential) AS Average_Potential,
    AVG(Dribbling) AS Average_Dribbling,
    AVG(Curve) AS Average_Curve,
    AVG(Free_Kick_Accuracy) AS Average_Free_Kick_Accuracy,
    AVG(Long_Pass) AS Average_Long_Pass,
    AVG(Ball_Control) AS Average_Ball_Control,
    AVG(Acceleration) AS Average_Acceleration,
    AVG(Sprint_Speed) AS Average_Sprint_Speed,
    AVG(Agility) AS Average_Agility,
    AVG(Reactions) AS Average_Reactions,
    AVG(Balance) AS Average_Balance,
    AVG(Shot_Power) AS Average_Shot_Power,
    AVG(Jumping) AS Average_Jumping,
    AVG(Stamina) AS Average_Stamina,
    AVG(Strength) AS Average_Strength,
    AVG(Long_Shot) AS Average_Long_Shot
FROM 
    Player_age 
GROUP BY 
    1
ORDER BY 
    1;

```



#### Identify the top 5 players who have shown the most improvement in their overall rating from their first appearance to their last appearance. Display the player's name, their initial overall rating, their final overall rating, and the improvement in their overall rating.
```sql
WITH Player_rank AS (
    SELECT 
        p.player_name, 
        pa.overall_rating, 
        pa.date,
        ROW_NUMBER() OVER (PARTITION BY p.player_name ORDER BY pa.date ASC) AS row_no,
        COUNT(*) OVER (PARTITION BY p.player_name) AS total_rows
    FROM 
        player p
    JOIN 
        player_attributes pa ON pa.player_api_id = p.player_api_id
    WHERE 
        pa.overall_rating IS NOT NULL
),
Initial_final_rating AS (
    SELECT 
        player_name, 
        MAX(CASE WHEN row_no = 1 THEN overall_rating ELSE 0 END) AS Initial_rating,
        MAX(CASE WHEN row_no = total_rows THEN overall_rating ELSE 0 END) AS Final_rating
    FROM 
        Player_rank
    GROUP BY 
        1
)
SELECT 
    *, 
    CONCAT(ROUND((Final_rating - Initial_rating) / Initial_rating * 100, 2), ' %') AS Improved_rating 
FROM 
    Initial_final_rating
ORDER BY 
    Improved_rating DESC
LIMIT 5;

```
