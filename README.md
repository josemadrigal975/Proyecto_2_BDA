# Proyecto_2_BDA

# Análisis de Canciones de Spotify

Este repositorio contiene un proyecto de análisis de datos basado en canciones de Spotify. Utiliza Python para extraer, transformar y analizar datos, generando visualizaciones útiles para identificar patrones y tendencias musicales.



## Descripción del Proyecto

Este proyecto se centra en analizar un dataset de Spotify que contiene información sobre canciones populares en diferentes países. Las principales metas incluyen:
1. Limpieza y transformación de los datos.
2. Cálculo de KPIs relevantes, como la cantidad de canciones rankeadas por mes y la popularidad promedio por artista.
3. Generación de visualizaciones para explorar patrones y tendencias.

---

## Características Principales

- **ETL (Extracción, Transformación y Carga):** Procesa datos desde un archivo CSV y los carga en una base de datos PostgreSQL.
- **Consultas Analíticas:** Calcula KPIs como la popularidad promedio y analiza patrones temporales y regionales.
- **Visualizaciones:** Genera gráficos claros y personalizados con `matplotlib`.

---

## Requisitos

Antes de ejecutar el proyecto, asegúrate de tener instaladas las siguientes herramientas:

- Python 3.8 o superior
- PostgreSQL
- Librerías Python (ver [Instalación](#instalación))

---

## Configuración de puertos de acuerdo a la información brindada en PostgreSQL
DB_USER = 'tu_usuario'
DB_PASSWORD = 'tu_contraseña'
DB_HOST = 'localhost'
DB_PORT = '5432'
DB_NAME = 'data_warehouse'

      import pandas as pd
    from sqlalchemy import create_engine

    # Configuración de conexión a la base de datos
    DB_USER = 'postgres'
    DB_PASSWORD = 'Base'
    DB_HOST = 'localhost'
    DB_PORT = '5432'
    DB_NAME = 'data_warehouse'

    DATABASE_URI = f'postgresql+psycopg2://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}'
    engine = create_engine(DATABASE_URI)

    def clean_data(raw_data):
        """
        Limpia los datos eliminando espacios en blanco y valores inconsistentes.
        """
        raw_data.columns = raw_data.columns.str.strip()
        for col in raw_data.select_dtypes(include=['object']).columns:
            raw_data[col] = raw_data[col].str.strip()
        
        raw_data.replace(['', ' ', 'null', 'NULL'], pd.NA, inplace=True)
        raw_data.dropna(how='all', inplace=True)
        
        print("Datos después de limpieza:")
        print(raw_data.head())
        
        return raw_data

    def extract_data(file_path):
        """
        Extrae datos del archivo CSV.
        """
        try:
            raw_data = pd.read_csv(file_path, encoding='utf-8', low_memory=False)
            print("Datos extraídos correctamente.")
        except UnicodeDecodeError:
            print("Error de codificación. Intentando con 'latin1'.")
            raw_data = pd.read_csv(file_path, encoding='latin1', low_memory=False)
        
        return clean_data(raw_data)

    def transform_data(raw_data):
        """
        Transforma los datos para ajustarse al esquema de dimensiones y hechos.
        """
        print(f"Columnas encontradas: {raw_data.columns}")
        
        raw_data['snapshot_date'] = pd.to_datetime(raw_data['snapshot_date'], errors='coerce')
        raw_data['Year'] = raw_data['snapshot_date'].dt.year
        raw_data['Month'] = raw_data['snapshot_date'].dt.month
        raw_data['Day'] = raw_data['snapshot_date'].dt.day
        raw_data['WeekDayName'] = raw_data['snapshot_date'].dt.day_name()
        raw_data['Quarter'] = raw_data['snapshot_date'].dt.quarter
        
        dim_date = raw_data[['snapshot_date', 'Year', 'Month', 'Day', 'WeekDayName', 'Quarter']].drop_duplicates()
        dim_date.rename(columns={'snapshot_date': 'DateValue'}, inplace=True)
        
        dim_country = raw_data[['country']].drop_duplicates().rename(columns={'country': 'CountryName'})
        
        dim_track = raw_data[['spotify_id', 'name', 'is_explicit', 'duration_ms']].drop_duplicates()
        dim_track.rename(columns={'spotify_id': 'SpotifyID', 'name': 'Nombre', 'is_explicit': 'IsExplicit', 'duration_ms': 'DurationMs'}, inplace=True)
        
        dim_ranking_type = pd.DataFrame({'RankingTypeName': ['Daily', 'Weekly']})
        dim_ranking_type.reset_index(inplace=True)
        dim_ranking_type.rename(columns={'index': 'RankTypeKey'}, inplace=True)
        
        dim_artist = raw_data[['artists']].drop_duplicates().rename(columns={'artists': 'ArtistName'})
        
        dim_album = raw_data[['album_name', 'album_release_date']].drop_duplicates()
        dim_album.rename(columns={'album_name': 'AlbumName', 'album_release_date': 'AlbumReleaseDate'}, inplace=True)
        
        dim_snapshot = raw_data[['snapshot_date']].drop_duplicates().rename(columns={'snapshot_date': 'SnapshotDate'})
        
        fact_ranking = raw_data[['daily_rank', 'daily_movement', 'weekly_movement', 'snapshot_date', 'country', 'spotify_id']].copy()
        fact_ranking.rename(columns={
            'daily_rank': 'DailyRank',
            'daily_movement': 'DailyMovement',
            'weekly_movement': 'WeeklyMovement',
            'snapshot_date': 'DateKey',
            'country': 'CountryKey',
            'spotify_id': 'TrackKey'
        }, inplace=True)
        
        fact_song_metrics = raw_data[['popularity', 'danceability', 'energy', 'key', 'loudness', 'mode', 'speechiness',
                                      'acousticness', 'instrumentalness', 'liveness', 'valence', 'tempo', 'time_signature',
                                      'spotify_id', 'artists', 'album_name', 'snapshot_date']].copy()
        fact_song_metrics.rename(columns={
            'popularity': 'Popularity',
            'danceability': 'Danceability',
            'energy': 'Energy',
            'key': 'Key',
            'loudness': 'Loudness',
            'mode': 'Mode',
            'speechiness': 'Speechiness',
            'acousticness': 'Acousticness',
            'instrumentalness': 'Instrumentalness',
            'liveness': 'Liveness',
            'valence': 'Valence',
            'tempo': 'Tempo',
            'time_signature': 'TimeSignature',
            'spotify_id': 'TrackKey',
            'artists': 'ArtistKey',
            'album_name': 'AlbumKey',
            'snapshot_date': 'SnapshotKey'
        }, inplace=True)

        print("Datos transformados correctamente.")
        return dim_date, dim_country, dim_track, dim_ranking_type, dim_artist, dim_album, dim_snapshot, fact_ranking, fact_song_metrics

    def load_data(engine, dim_date, dim_country, dim_track, dim_ranking_type, dim_artist, dim_album, dim_snapshot, fact_ranking, fact_song_metrics):
        """
        Carga las tablas de dimensiones y hechos en la base de datos.
        """
        dim_date.to_sql('DimDate', engine, if_exists='replace', index=False)
        dim_country.to_sql('DimCountry', engine, if_exists='replace', index=False)
        dim_track.to_sql('DimTrack', engine, if_exists='replace', index=False)
        dim_ranking_type.to_sql('DimRankingType', engine, if_exists='replace', index=False)
        dim_artist.to_sql('DimArtist', engine, if_exists='replace', index=False)
        dim_album.to_sql('DimAlbum', engine, if_exists='replace', index=False)
        dim_snapshot.to_sql('DimSnapshot', engine, if_exists='replace', index=False)
        fact_ranking.to_sql('FactRanking', engine, if_exists='replace', index=False)
        fact_song_metrics.to_sql('FactSongMetrics', engine, if_exists='replace', index=False)
        
        print("Datos cargados correctamente en la base de datos.")

    def etl_pipeline(file_path):
        """
        Ejecuta el pipeline ETL completo.
        """
        raw_data = extract_data(file_path)
        dim_date, dim_country, dim_track, dim_ranking_type, dim_artist, dim_album, dim_snapshot, fact_ranking, fact_song_metrics = transform_data(raw_data)
        load_data(engine, dim_date, dim_country, dim_track, dim_ranking_type, dim_artist, dim_album, dim_snapshot, fact_ranking, fact_song_metrics)

    if __name__ == "__main__":
        file_path = r'c:\Archivos\universal_top_spotify_songs.csv'
        etl_pipeline(file_path)
        

