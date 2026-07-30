import React, { useCallback, useEffect, useState } from 'react';
import { View, ScrollView, StyleSheet, Text, ActivityIndicator, RefreshControl } from 'react-native';
import { StatusBar } from 'expo-status-bar';
import * as Location from 'expo-location';
import { useFonts, SpaceGrotesk_700Bold, SpaceGrotesk_500Medium } from '@expo-google-fonts/space-grotesk';
import { JetBrainsMono_400Regular, JetBrainsMono_500Medium } from '@expo-google-fonts/jetbrains-mono';

import BackgroundSky from './src/components/BackgroundSky';
import SearchBar from './src/components/SearchBar';
import CurrentWeather from './src/components/CurrentWeather';
import HourlyForecast from './src/components/HourlyForecast';
import DailyForecast from './src/components/DailyForecast';
import { fetchForecast, reverseGeocode } from './src/api/weather';
import { getWeatherInfo, twilightPaletteKey } from './src/utils/weatherCodes';
import { colors, type, space } from './src/theme';

// Default location shown before the user grants location access or searches.
const DEFAULT_PLACE = { name: 'New York', latitude: 40.7128, longitude: -74.006 };
const TWILIGHT_WINDOW_MIN = 35;

export default function App() {
  const [fontsLoaded] = useFonts({
    SpaceGrotesk_700Bold,
    SpaceGrotesk_500Medium,
    JetBrainsMono_400Regular,
    JetBrainsMono_500Medium,
  });

  const [place, setPlace] = useState(DEFAULT_PLACE);
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  const [locating, setLocating] = useState(false);
  const [errorMsg, setErrorMsg] = useState(null);

  const loadWeather = useCallback(async (lat, lon, isRefresh = false) => {
    try {
      isRefresh ? setRefreshing(true) : setLoading(true);
      setErrorMsg(null);
      const forecast = await fetchForecast(lat, lon);
      setData(forecast);
    } catch (e) {
      setErrorMsg("Couldn't load weather. Pull down to try again.");
    } finally {
      setLoading(false);
      setRefreshing(false);
    }
  }, []);

  useEffect(() => {
    (async () => {
      // Try device location first; fall back to the default city silently.
      try {
        setLocating(true);
        const { status } = await Location.requestForegroundPermissionsAsync();
        if (status === 'granted') {
          const pos = await Location.getCurrentPositionAsync({});
          const { latitude, longitude } = pos.coords;
          const place = await reverseGeocode(latitude, longitude);
          setPlace({
            name: place?.name || 'Current location',
            admin1: place?.admin1,
            country: place?.country,
            latitude,
            longitude,
          });
          await loadWeather(latitude, longitude);
          setLocating(false);
          return;
        }
      } catch {
        // fall through to default
      }
      setLocating(false);
      await loadWeather(DEFAULT_PLACE.latitude, DEFAULT_PLACE.longitude);
    })();
  }, []);

  const handleSelectPlace = (p) => {
    setPlace(p);
    loadWeather(p.latitude, p.longitude);
  };

  const handleUseLocation = async () => {
    try {
      setLocating(true);
      const { status } = await Location.requestForegroundPermissionsAsync();
      if (status !== 'granted') {
        setLocating(false);
        return;
      }
      const pos = await Location.getCurrentPositionAsync({});
      const { latitude, longitude } = pos.coords;
      const geo = await reverseGeocode(latitude, longitude);
      setPlace({
        name: geo?.name || 'Current location',
        admin1: geo?.admin1,
        country: geo?.country,
        latitude,
        longitude,
      });
      await loadWeather(latitude, longitude);
    } finally {
      setLocating(false);
    }
  };

  const handleRefresh = () => {
    if (place) loadWeather(place.latitude, place.longitude, true);
  };

  if (!fontsLoaded) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color="#fff" />
      </View>
    );
  }

  const current = data?.current;
  const daily = data?.daily;
  const hourly = data?.hourly;

  // Work out which sky palette + orb position to show, based on real
  // sunrise/sunset for the selected place rather than a fixed day/night flip.
  let paletteKey = 'day_clear';
  let orbHeight = 0.5;
  let icon = 'partly-sunny';
  let description = 'Loading…';

  if (current && daily) {
    const now = new Date(current.time);
    const sunrise = new Date(daily.sunrise[0]);
    const sunset = new Date(daily.sunset[0]);
    const minsTo = (a, b) => Math.abs(a.getTime() - b.getTime()) / 60000;
    const isDay = !!current.is_day;
    const info = getWeatherInfo(current.weather_code, isDay);

    if (minsTo(now, sunrise) < TWILIGHT_WINDOW_MIN) {
      paletteKey = twilightPaletteKey(true);
    } else if (minsTo(now, sunset) < TWILIGHT_WINDOW_MIN) {
      paletteKey = twilightPaletteKey(false);
    } else {
      paletteKey = info.paletteKey;
    }
    icon = info.icon;
    description = info.description;

    if (isDay) {
      const dayLen = sunset.getTime() - sunrise.getTime();
      const frac = Math.min(1, Math.max(0, (now.getTime() - sunrise.getTime()) / dayLen));
      orbHeight = Math.sin(Math.PI * frac); // 0 at sunrise/sunset, 1 at solar noon
    } else {
      orbHeight = 0.15;
    }
  }

  return (
    <View style={styles.flex}>
      <StatusBar style="light" />
      <BackgroundSky paletteKey={paletteKey} isDay={!!current?.is_day} orbHeight={orbHeight} />

      <ScrollView
        style={styles.flex}
        contentContainerStyle={styles.scrollContent}
        refreshControl={
          <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} tintColor="#fff" />
        }
        showsVerticalScrollIndicator={false}
      >
        <SearchBar onSelectPlace={handleSelectPlace} onUseLocation={handleUseLocation} locating={locating} />

        {loading ? (
          <View style={styles.loadingBlock}>
            <ActivityIndicator size="large" color="#fff" />
          </View>
        ) : errorMsg ? (
          <View style={styles.loadingBlock}>
            <Text style={styles.errorText}>{errorMsg}</Text>
          </View>
        ) : (
          <>
            <CurrentWeather
              placeName={[place.name, place.admin1].filter(Boolean).join(', ')}
              current={current}
              todayHigh={daily?.temperature_2m_max?.[0]}
              todayLow={daily?.temperature_2m_min?.[0]}
              icon={icon}
              description={description}
            />
            <HourlyForecast hourly={hourly} currentIsoHour={current?.time?.slice(0, 13) + ':00'} />
            <DailyForecast daily={daily} />
          </>
        )}
      </ScrollView>
    </View>
  );
}

const styles = StyleSheet.create({
  flex: { flex: 1 },
  scrollContent: { paddingTop: 60, paddingBottom: 40 },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', backgroundColor: '#111C36' },
  loadingBlock: { marginTop: 100, alignItems: 'center' },
  errorText: { color: colors.text, fontFamily: type.body, fontSize: 15, textAlign: 'center', paddingHorizontal: space.xl },
});
