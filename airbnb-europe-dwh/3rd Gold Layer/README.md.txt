create database airbnbDB;

use airbnbDB;
go

-- Dimension: City
CREATE TABLE dim_city (
    city_id INT PRIMARY KEY,
    city VARCHAR(100)
);

-- Dimension: Room
CREATE TABLE dim_room (
    room_id INT PRIMARY KEY,
    room_type VARCHAR(100),
    room_shared BIT,
    room_private BIT,
    person_capacity DECIMAL(5,2),
    bedrooms INT
);

-- Dimension: Host
CREATE TABLE dim_host (
    host_id INT PRIMARY KEY,
    host_is_superhost BIT,
    multi BIT,
    biz BIT
);

-- Dimension: Location
CREATE TABLE dim_location (
    location_id INT PRIMARY KEY,
    lng DECIMAL(10,6),
    lat DECIMAL(10,6),
    dist DECIMAL(10,6),
    metro_dist DECIMAL(10,6)
);

-- Dimension: Attraction
CREATE TABLE dim_attraction (
    attraction_id INT PRIMARY KEY,
    attr_index DECIMAL(15,6),
    attr_index_norm DECIMAL(15,6),
    rest_index DECIMAL(15,6),
    rest_index_norm DECIMAL(15,6)
);

-- Dimension: Date
CREATE TABLE dim_date (
    date_id INT PRIMARY KEY,
    is_weekend BIT,
    period_type VARCHAR(50)
);

USE airbnbDB;
GO

DELETE FROM dim_city;
DELETE FROM dim_room;
DELETE FROM dim_host;
DELETE FROM dim_location;
DELETE FROM dim_attraction;
DELETE FROM dim_date;
DROP TABLE IF EXISTS fact_listings;
GO

INSERT INTO dim_city (city_id, city)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY city) AS city_id,
    city
FROM Airbnb_Combined_Data;
GO

INSERT INTO dim_room (room_id, room_type, room_shared, room_private, person_capacity, bedrooms)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY room_type, room_shared, room_private, person_capacity, bedrooms) AS room_id,
    room_type, room_shared, room_private, person_capacity, bedrooms
FROM Airbnb_Combined_Data;
GO

INSERT INTO dim_host (host_id, host_is_superhost, multi, biz)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY host_is_superhost, multi, biz) AS host_id,
    host_is_superhost, multi, biz
FROM Airbnb_Combined_Data;
GO

INSERT INTO dim_location (location_id, lng, lat, dist, metro_dist)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY lng, lat, dist, metro_dist) AS location_id,
    TRY_CAST(lng AS DECIMAL(10,6)),
    TRY_CAST(lat AS DECIMAL(10,6)),
    dist,
    metro_dist
FROM Airbnb_Combined_Data;
GO

INSERT INTO dim_attraction (attraction_id, attr_index, attr_index_norm, rest_index, rest_index_norm)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY attr_index, attr_index_norm, rest_index, rest_index_norm) AS attraction_id,
    attr_index, attr_index_norm, rest_index, rest_index_norm
FROM Airbnb_Combined_Data;
GO

INSERT INTO dim_date (date_id, is_weekend, period_type)
SELECT DISTINCT
    DENSE_RANK() OVER (ORDER BY day_type) AS date_id,
    CASE WHEN day_type = 'weekends' THEN 1 ELSE 0 END AS is_weekend,
    day_type AS period_type
FROM Airbnb_Combined_Data;
GO


CREATE TABLE fact_listings (
    listing_id                   INT IDENTITY(1,1) PRIMARY KEY,
    city_id                      INT NOT NULL,
    room_id                      INT NOT NULL,
    host_id                      INT NOT NULL,
    location_id                  INT NOT NULL,
    attraction_id                INT NOT NULL,
    date_id                      INT NOT NULL,

    price                        FLOAT,
    cleanliness_rating           FLOAT,
    guest_satisfaction_overall   FLOAT
);
GO

INSERT INTO fact_listings (
    city_id, room_id, host_id, location_id, attraction_id, date_id,
    price, cleanliness_rating, guest_satisfaction_overall
)
SELECT
    DENSE_RANK() OVER (ORDER BY city)                                                              AS city_id,
    DENSE_RANK() OVER (ORDER BY room_type, room_shared, room_private, person_capacity, bedrooms)     AS room_id,
    DENSE_RANK() OVER (ORDER BY host_is_superhost, multi, biz)                                       AS host_id,
    DENSE_RANK() OVER (ORDER BY lng, lat, dist, metro_dist)                                          AS location_id,
    DENSE_RANK() OVER (ORDER BY attr_index, attr_index_norm, rest_index, rest_index_norm)             AS attraction_id,
    DENSE_RANK() OVER (ORDER BY day_type)                                                            AS date_id,
    realSum,
    cleanliness_rating,
    guest_satisfaction_overall
FROM Airbnb_Combined_Data;
GO

SELECT COUNT(*) AS fact_row_count FROM fact_listings;
SELECT COUNT(*) AS staging_row_count FROM Airbnb_Combined_Data;
GO