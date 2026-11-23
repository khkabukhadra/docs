# Filter Page — handoff

Client-side filter/sort for the freelancers list (`src/app/expertises/[id]/freelancers.tsx`). Opens as a modal (confirm with the team which existing screen's modal presentation to match — there's no route literally named "Contracts" in this repo; closest candidates are `(tabs)/pings/index.tsx` and `(tabs)/responses.tsx`).

No backend changes, no new dependencies. All fields used (`hourlyRate`, `ratingSum`/`interactionCount`, `latitude`/`longitude`) already exist on `FreelancerResponse`.

---

## 1. `src/lib/freelancerFilters.ts` (new)

```ts
import type { FreelancerResponse } from '@/api/types';

export type SortBy = 'rating' | 'price' | 'distance';
export type SortDir = 'asc' | 'desc';

export type FilterOptions = {
  minRating?: number;
  maxPrice?: number;
  sortBy: SortBy;
  sortDir: SortDir;
};

export const DEFAULT_FILTERS: FilterOptions = {
  sortBy: 'rating',
  sortDir: 'desc',
};

export function getRating(freelancer: FreelancerResponse): number {
  if (freelancer.interactionCount === 0) return 0;
  return freelancer.ratingSum / freelancer.interactionCount;
}

export function applyFilters(
  freelancers: FreelancerResponse[],
  options: FilterOptions,
  distanceById?: Map<string, number>,
): FreelancerResponse[] {
  let result = freelancers;

  if (options.minRating !== undefined) {
    result = result.filter((f) => getRating(f) >= options.minRating!);
  }

  if (options.maxPrice !== undefined) {
    result = result.filter((f) => f.hourlyRate <= options.maxPrice!);
  }

  const dir = options.sortDir === 'asc' ? 1 : -1;

  result = [...result].sort((a, b) => {
    switch (options.sortBy) {
      case 'rating':
        return (getRating(a) - getRating(b)) * dir;
      case 'price':
        return (a.hourlyRate - b.hourlyRate) * dir;
      case 'distance': {
        const da = distanceById?.get(a.id) ?? Infinity;
        const db = distanceById?.get(b.id) ?? Infinity;
        return (da - db) * dir;
      }
    }
  });

  return result;
}

export function isDefaultFilters(options: FilterOptions): boolean {
  return (
    options.minRating === undefined &&
    options.maxPrice === undefined &&
    options.sortBy === DEFAULT_FILTERS.sortBy &&
    options.sortDir === DEFAULT_FILTERS.sortDir
  );
}
```

---

## 2. `src/hooks/useCurrentLocation.ts` (new)

One-shot location fetch for the whole list — distinct from `useDistanceTo.ts`, which re-fetches per card on focus and doesn't scale to sorting a list.

```ts
// useCurrentLocation.ts
import { useCallback, useEffect, useState } from 'react';
import { getCurrentLocation, LocationResult, Coords } from '../lib/location';

type State =
  | { status: 'loading' }
  | { status: 'ready'; coords: Coords }
  | { status: 'unavailable'; reason: Extract<LocationResult, { ok: false }>['reason'] };

export function useCurrentLocation(enabled: boolean) {
  const [state, setState] = useState<State>({ status: 'loading' });

  const refresh = useCallback(async () => {
    if (!enabled) return;
    setState({ status: 'loading' });
    const res = await getCurrentLocation();
    setState(
      res.ok
        ? { status: 'ready', coords: res.coords }
        : { status: 'unavailable', reason: res.reason },
    );
  }, [enabled]);

  useEffect(() => {
    if (enabled) refresh();
  }, [enabled, refresh]);

  return { state, refresh };
}
```

---

## 3. `src/app/expertises/[id]/_layout.tsx` (new)

Doesn't exist yet at this route level — needed to present `filters` as a modal.

```tsx
import { Stack } from 'expo-router';

export default function ExpertiseFreelancersLayout() {
  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Screen name="freelancers" />
      <Stack.Screen name="filters" options={{ presentation: 'modal' }} />
    </Stack>
  );
}
```

---

## 4. `src/components/FilterButton.tsx` (new)

```tsx
import { Pressable, Text, View, StyleSheet } from 'react-native';
import { router } from 'expo-router';

type FilterButtonProps = {
  expertiseId: string;
  active: boolean;
  params: Record<string, string>;
};

export function FilterButton({ expertiseId, active, params }: FilterButtonProps) {
  return (
    <Pressable
      style={styles.button}
      onPress={() => router.push({ pathname: `/expertises/${expertiseId}/filters`, params })}>
      <Text style={styles.label}>Filter</Text>
      {active && <View style={styles.dot} />}
    </Pressable>
  );
}

const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 6,
    paddingVertical: 8,
    paddingHorizontal: 14,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: 'rgba(191, 64, 255, 0.4)',
    backgroundColor: 'rgba(17, 10, 31, 0.5)',
  },
  label: {
    color: '#BF40FF',
    fontSize: 14,
    fontWeight: '600',
  },
  dot: {
    width: 6,
    height: 6,
    borderRadius: 3,
    backgroundColor: '#BF40FF',
  },
});
```

---

## 5. `src/app/expertises/[id]/filters.tsx` (new)

```tsx
import { useState } from 'react';
import { Pressable, ScrollView, Text, TextInput, View, StyleSheet } from 'react-native';
import { router, useLocalSearchParams } from 'expo-router';
import { SafeAreaView } from 'react-native-safe-area-context';

import type { SortBy, SortDir } from '@/lib/freelancerFilters';

const SORT_OPTIONS: { value: SortBy; label: string }[] = [
  { value: 'rating', label: 'Rating' },
  { value: 'price', label: 'Price' },
  { value: 'distance', label: 'Distance' },
];

export default function FiltersScreen() {
  const { id, minRating, maxPrice, sortBy, sortDir } = useLocalSearchParams<{
    id: string;
    minRating?: string;
    maxPrice?: string;
    sortBy?: SortBy;
    sortDir?: SortDir;
  }>();

  const [minRatingInput, setMinRatingInput] = useState(minRating ?? '');
  const [maxPriceInput, setMaxPriceInput] = useState(maxPrice ?? '');
  const [selectedSortBy, setSelectedSortBy] = useState<SortBy>(sortBy ?? 'rating');
  const [selectedSortDir, setSelectedSortDir] = useState<SortDir>(sortDir ?? 'desc');

  const apply = () => {
    router.push({
      pathname: `/expertises/${id}/freelancers`,
      params: {
        ...(minRatingInput ? { minRating: minRatingInput } : {}),
        ...(maxPriceInput ? { maxPrice: maxPriceInput } : {}),
        sortBy: selectedSortBy,
        sortDir: selectedSortDir,
      },
    });
  };

  const clear = () => {
    setMinRatingInput('');
    setMaxPriceInput('');
    setSelectedSortBy('rating');
    setSelectedSortDir('desc');
    router.push({ pathname: `/expertises/${id}/freelancers` });
  };

  return (
    <SafeAreaView
      style={{
        flex: 1,
        backgroundColor: '#241B32',
        experimental_backgroundImage: 'radial-gradient(circle at 0% 0%, #83739A, transparent 60%)',
      }}>
      <ScrollView contentContainerStyle={{ padding: 16 }}>
        <Text style={styles.title}>Filter & Sort</Text>

        <Text style={styles.sectionLabel}>Minimum rating</Text>
        <TextInput
          style={styles.input}
          placeholder="e.g. 4.0"
          placeholderTextColor="rgba(255,255,255,0.4)"
          keyboardType="decimal-pad"
          value={minRatingInput}
          onChangeText={setMinRatingInput}
        />

        <Text style={styles.sectionLabel}>Max price (per hour)</Text>
        <TextInput
          style={styles.input}
          placeholder="e.g. 500"
          placeholderTextColor="rgba(255,255,255,0.4)"
          keyboardType="number-pad"
          value={maxPriceInput}
          onChangeText={setMaxPriceInput}
        />

        <Text style={styles.sectionLabel}>Sort by</Text>
        <View style={styles.row}>
          {SORT_OPTIONS.map((opt) => (
            <Pressable
              key={opt.value}
              style={[styles.chip, selectedSortBy === opt.value && styles.chipActive]}
              onPress={() => setSelectedSortBy(opt.value)}>
              <Text style={[styles.chipLabel, selectedSortBy === opt.value && styles.chipLabelActive]}>
                {opt.label}
              </Text>
            </Pressable>
          ))}
        </View>

        <Text style={styles.sectionLabel}>Direction</Text>
        <View style={styles.row}>
          {(['desc', 'asc'] as SortDir[]).map((dir) => (
            <Pressable
              key={dir}
              style={[styles.chip, selectedSortDir === dir && styles.chipActive]}
              onPress={() => setSelectedSortDir(dir)}>
              <Text style={[styles.chipLabel, selectedSortDir === dir && styles.chipLabelActive]}>
                {dir === 'desc' ? 'High to low' : 'Low to high'}
              </Text>
            </Pressable>
          ))}
        </View>

        <Pressable style={styles.applyButton} onPress={apply}>
          <Text style={styles.applyLabel}>Apply</Text>
        </Pressable>
        <Pressable style={styles.clearButton} onPress={clear}>
          <Text style={styles.clearLabel}>Clear filters</Text>
        </Pressable>
      </ScrollView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  title: {
    fontWeight: 'bold',
    fontSize: 28,
    fontFamily: 'Rubik',
    color: '#FFF',
    marginBottom: 20,
  },
  sectionLabel: {
    color: '#BF40FF',
    fontSize: 13,
    marginBottom: 8,
    marginTop: 16,
  },
  input: {
    backgroundColor: 'rgba(17, 10, 31, 0.5)',
    borderWidth: 1,
    borderColor: 'rgba(191, 64, 255, 0.4)',
    borderRadius: 10,
    paddingVertical: 10,
    paddingHorizontal: 14,
    color: '#FFF',
    fontSize: 16,
  },
  row: {
    flexDirection: 'row',
    gap: 10,
    flexWrap: 'wrap',
  },
  chip: {
    paddingVertical: 8,
    paddingHorizontal: 16,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: 'rgba(191, 64, 255, 0.4)',
    backgroundColor: 'rgba(17, 10, 31, 0.5)',
  },
  chipActive: {
    backgroundColor: '#BF40FF',
    borderColor: '#BF40FF',
  },
  chipLabel: {
    color: '#BF40FF',
    fontSize: 14,
  },
  chipLabelActive: {
    color: '#FFF',
    fontWeight: '600',
  },
  applyButton: {
    marginTop: 32,
    backgroundColor: '#BF40FF',
    borderRadius: 12,
    paddingVertical: 14,
    alignItems: 'center',
  },
  applyLabel: {
    color: '#FFF',
    fontWeight: 'bold',
    fontSize: 16,
  },
  clearButton: {
    marginTop: 12,
    paddingVertical: 14,
    alignItems: 'center',
  },
  clearLabel: {
    color: 'rgba(255,255,255,0.6)',
    fontSize: 14,
  },
});
```

---

## 6. `src/app/expertises/[id]/freelancers.tsx` (edit — full replacement)

```tsx
import { useEffect, useMemo, useState } from 'react'
import { ActivityIndicator, FlatList, Pressable, Text, View } from 'react-native'
import { useLocalSearchParams } from 'expo-router'
import { SafeAreaView } from 'react-native-safe-area-context'

import type { FreelancerResponse, FreelancersResponse } from '@/api/types'
import { FreelancerCard } from '@/components/FreelancerCard'
import { FilterButton } from '@/components/FilterButton'
import { getAsync } from '@/api/client'
import {
  applyFilters,
  isDefaultFilters,
  DEFAULT_FILTERS,
  FilterOptions,
  SortBy,
  SortDir,
} from '@/lib/freelancerFilters'
import { useCurrentLocation } from '@/hooks/useCurrentLocation'
import { distanceMeters } from '@/lib/geo'
import { openLocationSettings } from '@/lib/location'

export default function ExpertiseScreeen() {
  const { id, minRating, maxPrice, sortBy, sortDir } = useLocalSearchParams<{
    id: string;
    minRating?: string;
    maxPrice?: string;
    sortBy?: SortBy;
    sortDir?: SortDir;
  }>();
  const [freelancers, setFreelancers] = useState<FreelancerResponse[]>([]);
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  const filters: FilterOptions = {
    minRating: minRating ? Number(minRating) : undefined,
    maxPrice: maxPrice ? Number(maxPrice) : undefined,
    sortBy: sortBy ?? DEFAULT_FILTERS.sortBy,
    sortDir: sortDir ?? DEFAULT_FILTERS.sortDir,
  };

  const needsLocation = filters.sortBy === 'distance';
  const { state: locationState, refresh: refreshLocation } = useCurrentLocation(needsLocation);

  useEffect(() => {
    const load = async () => {
      try {
        const freelancersResponse: FreelancersResponse = await getAsync(`api/freelancers?expertiseId=${id}`);
        setFreelancers(freelancersResponse.items)
      }
      catch (e) {
        setError(e instanceof Error ? e.message : 'Failed to load freelancer');
      }
      finally {
        setLoading(false);
      }
    };
    load();
  }, [id]);

  const distanceById = useMemo(() => {
    if (locationState.status !== 'ready') return undefined;
    const map = new Map<string, number>();
    for (const f of freelancers) {
      map.set(f.id, distanceMeters(locationState.coords, { latitude: f.latitude, longitude: f.longitude }));
    }
    return map;
  }, [freelancers, locationState]);

  const visibleFreelancers = useMemo(
    () => applyFilters(freelancers, filters, distanceById),
    [freelancers, filters, distanceById],
  );

  if (loading) return <View><ActivityIndicator /></View>;
  if (error) return <Text>{error}</Text>;

  return (
    <SafeAreaView
      style={{
        flex: 1,
        backgroundColor: '#241B32',
        experimental_backgroundImage: 'radial-gradient(circle at 0% 0%, #83739A, transparent 60%)',
      }}>
      <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', padding: 16 }}>
        <Text style={{ fontWeight: 'bold', fontSize: 39, fontFamily: 'Rubik', color: '#FFF' }}>
          Freelancers
        </Text>
        <FilterButton
          expertiseId={id}
          active={!isDefaultFilters(filters)}
          params={{
            ...(minRating ? { minRating } : {}),
            ...(maxPrice ? { maxPrice } : {}),
            sortBy: filters.sortBy,
            sortDir: filters.sortDir,
          }}
        />
      </View>

      {needsLocation && locationState.status === 'unavailable' && (
        <DistanceWarning reason={locationState.reason} refresh={refreshLocation} />
      )}

      <FlatList
        data={visibleFreelancers}
        keyExtractor={(f) => f.id.toString()}
        contentContainerStyle={{ padding: 16 }}
        ItemSeparatorComponent={() => <View style={{ height: 22 }} />}
        renderItem={({ item }) => (
          <FreelancerCard freelancer={item}/>
        )}
      />
    </SafeAreaView>
  );
}

function DistanceWarning({ reason, refresh }: { reason: string; refresh: () => void }) {
  const needsSettings = reason === 'blocked' || reason === 'services-off';
  return (
    <Pressable
      onPress={needsSettings ? openLocationSettings : refresh}
      style={{ paddingHorizontal: 16, paddingBottom: 8 }}>
      <Text style={{ color: '#BF40FF', fontSize: 13 }}>
        {needsSettings ? 'Enable location to sort by distance' : 'Location unavailable — tap to retry'}
      </Text>
    </Pressable>
  );
}
```

---

## Notes for whoever picks this up

- `FreelancerCard.tsx` already imports `getDistanceDisplay` but never renders it — distance isn't shown on the card yet, independent of this feature.
- "Modal that replaces the current Contracts page" — there's no route literally named "Contracts"; nail down whether that means `(tabs)/pings/index.tsx` or `(tabs)/responses.tsx` before wiring the modal's exact presentation/navigation behavior.
- Everything here is untested against the real API — verify `api/freelancers?expertiseId=` actually returns all freelancers for the expertise (not paginated) before relying on client-side filtering being complete.
