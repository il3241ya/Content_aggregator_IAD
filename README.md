# Агрегатор контента YouTube


## Введение

В ходе данного проекта был разработан агрегатор контента с видеохостинга YouTube. Данная программа используется владельцем канала(ов) для сбора статистики, и ее последующего мониторинга. Это полезно для эффективного управления контентом, привлечения аудитории, анализа популярности контента, а так же для выбора наилучшей стратегии монетезации.

## Функциональные требования

К данному проекту предъявлялись следующие функциональные требования:

1. Управление контентом:
    1. Добавление информации о каналах и привязанных к ним видео.
    2. Связывание нескольких каналов в группы через отдельную таблицу.
2. Аналитика:
    1. Составить полноценной аналитики для канала, как в целом, так и за определённый промежуток времени.
    2. Сбор, хранение, просмотр собранных статистических данных.
3. Монетизация:
    1. Отслеживание доходности канала.
    2. Отслеживание монетизации видео.
4. Отслеживание данных видео, комментариев.
    1. Отслеживание информации о видео и каналах.
    2. Отслеживание комментариев на канале.
5. Фильтрация:
    1. Возможность фильтрации видео и каналов по различным параметрам, таким как категория, дата, количество просмотров и т.д

## Нефункциональные требования

К данному проекту предъявлялись следующие нефункциональные требования:

1. Удобство использования и сопровождения. Данные должны быть разнесены по различным таблицам, согласно тематики, соотношений с другими элементами, это обеспечит удобство обращения к ним.

## Er-диаграмма

![Final.drawio.png](/Final.drawio.png)

## Нормализация

Таблица Channel имела поле AssociatedChannels, содержащее набор каналов ассоциированных с данным. Так как 1НФ требует атомарности значений каждого поля, из таблицы Channels было удалено поле AssociatedChannels, вместо него была добавлена таблица AssociatedChannels содержащая два поля: ID основного канала, и ID канала ассоциированного с ним.

## Схема

```markdown
Table: Channel
- ChannelID (PrimaryKey)
- Name
- Description
- CreationDate
- Status
- ChannelImageURL
- ChannelCoverURL

Table: AssocietedChannels
- MainChannelID (PrimaryKey, ForeignKey: Channel.ChannelID)
- AssocietedChannelID (PrimaryKey, ForeignKey: Channel.ChannelID)

Table: ChannelStatistics
- ChannelID (PrimaryKey, ForeignKey: Channel.ChannelID)
- RequestTime
- SubscribersCount
- TotalViews
- TotalComments
- VideosCount
- LikesCount
- DislikesCount
- ChannelVisitsStats
- Revenue

Table: Video
- VideoID (PrimaryKey)
- RequestTime
- ViewsCount
- LikesCount
- Title
- DislikesCount
- ChannelID (ForeignKey: Channel.ChannelID)
- CommentsCount

Table: VideoDetails
- VideoID (PrimaryKey, ForeignKey: Video.VideoID)
- Duration
- Tags
- UploadDate
- MonetizationInfo
- CopyrightStatus
- CategoryID (ForeignKey: VideoCategory.CategoryID)

Table: VideoComments
- CommentID (PrimaryKey)
- VideoID (ForeignKey: Video.VideoID)
- CommentText
- CommentDate
- LikesCount
- DislikesCount

Table: VideoCategory
- CategoryID (PrimaryKey)
- CategoryName
```

## SQL-DDL скрипты

```sql
CREATE TABLE IF NOT EXISTS Channel (
	ChannelID SERIAL PRIMARY KEY,
	Name VARCHAR(30) NOT NULL,
	Description TEXT,
	Status BOOL DEFAULT TRUE,
	CreationDate TIMESTAMPTZ NOT NULL DEFAULT now(),
	ChannelImageURL VARCHAR(200),
	ChannelCoverURL VARCHAR(200)
);

CREATE TABLE IF NOT EXISTS AssocietedChannels (
	MainChannelID SERIAL REFERENCES Channel(ChannelID),
	AssocietedChannelID SERIAL REFERENCES Channel(ChannelID),
	CHECK(MainChannelID != AssocietedChannelID),
	PRIMARY KEY (MainChannelID, AssocietedChannelID)
);

CREATE TABLE IF NOT EXISTS ChannelStatistics (
	ChannelID SERIAL PRIMARY KEY REFERENCES Channel(ChannelID),
	RequestTime TIMESTAMPTZ DEFAULT now(),
	SubscribersCount BIGINT NOT NULL DEFAULT 0,
	TotalViews BIGINT NOT NULL DEFAULT 0,
	TotalComments BIGINT NOT NULL DEFAULT 0,
	VideosCount BIGINT NOT NULL DEFAULT 0,
	LikesCount BIGINT NOT NULL DEFAULT 0,
	DislikesCount BIGINT NOT NULL DEFAULT 0,
	CHECK(TotalComments <= TotalViews),
	CHECK(LikesCount + DislikesCount <= TotalViews)
);

CREATE TABLE IF NOT EXISTS Video (
	VideoID SERIAL PRIMARY KEY,
	ChannelID SERIAL REFERENCES Channel(ChannelID),
	RequestTime TIMESTAMPTZ DEFAULT now(),
	Title VARCHAR(70) NOT NULL,
	UploadDate TIMESTAMPTZ NOT NULL,
	ViewsCount BIGINT NOT NULL DEFAULT 0,
	LikesCount BIGINT NOT NULL DEFAULT 0,
	DislikesCount BIGINT NOT NULL DEFAULT 0,
	CommentsCount BIGINT NOT NULL DEFAULT 0
);

CREATE TABLE IF NOT EXISTS VideoCategory (
	CategoryID SERIAL PRIMARY KEY,
	Name VARCHAR(50) NOT NULL
);

CREATE TABLE IF NOT EXISTS VideoDetails (
	VideoID SERIAL PRIMARY KEY REFERENCES Video(VideoID),
	CategoryID SERIAL REFERENCES VideoCategory(CategoryID),
	Duration TIME NOT NULL DEFAULT '0:0:0',
	Tags TEXT,
	MonetizationInfo TEXT,
	CopyrightStatus TEXT,
	CHECK(Duration <= '10:00:00')
);

CREATE TABLE IF NOT EXISTS VideoComments (
	CommentID SERIAL PRIMARY KEY,
	VideoID SERIAL REFERENCES Video(VideoID),
	CommentText TEXT NOT NULL,
	CommentDate TIMESTAMPTZ NOT NULL DEFAULT now(),
	LikesCount BIGINT NOT NULL DEFAULT 0,
	DislikesCount BIGINT NOT NULL DEFAULT 0
);
```

## SQL-DML скрипты

```sql
-- Preperation stage
INSERT INTO VideoCategory(Name) VALUES ('Science'), ('Entertainment');

-- Create new channel record (videos + channel info)
INSERT INTO Channel(Name, Description, CreationDate) VALUES ('Amogus', 'Amomogus', '1.12.2020');
INSERT INTO Video(ChannelID, Title, UploadDate, ViewsCount, LikesCount) 
VALUES (1, 'Amogus', '2.12.2020', 1000, 1000),
	   (1, 'Amo', '3.12.2020', 1000, 1000),
	   (1, 'AAAAA', '4.12.2020', 1000, 1000);
INSERT INTO VideoDetails(VideoID, CategoryID) 
SELECT videoid, videoid % 2 + 1 FROM Video;
INSERT INTO ChannelStatistics 
SELECT ch.ChannelID, '1.12.2020', 1000, (SELECT SUM(ViewsCount) FROM Video v WHERE v.ChannelID = ch.ChannelID),
	   0, (SELECT COUNT(*) FROM Video v WHERE v.ChannelID = ch.ChannelID), 
	   (SELECT SUM(LikesCount) FROM Video v WHERE v.ChannelID = ch.ChannelID),
	   (SELECT SUM(dislikesCount) FROM Video v WHERE v.ChannelID = ch.ChannelID)
FROM Channel ch;

-- Collect info about specific channel
SELECT RequestTime as time, TotalViews, LikesCount, DislikesCount 
FROM ChannelStatistics
WHERE ChannelID = 1;

-- Collect info about specific video
SELECT RequestTime as time, ViewsCount, LikesCount, DislikesCount 
FROM Video
WHERE ChannelID = 1;
```