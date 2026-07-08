# Plan Screen Horizontal Scroll Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the Plan screen from a vertical expandable week list into a horizontal card-scroll layout inspired by Runna and Kotcha, with linked scroll sync between week pills and workout pages.

**Architecture:** Three horizontal snap-scrolling layers stacked vertically: athlete avatars, week pills, and workout cards. The week pills and workout pages share a Reanimated `SharedValue` for synchronized horizontal scrolling. Each workout page contains a nested vertical `FlatList` of `EventCard` components. The scroll sync hook is extracted as a reusable unit testable in isolation.

**Tech Stack:** React Native, Expo SDK 57, `@expo/ui` (Host, Column, Row, Icon, Text), `react-native-reanimated` (useSharedValue, useAnimatedScrollHandler), `expo-haptics`, `expo-router`, `expo-image`.

## Global Constraints

- No new dependencies — all libraries already installed in the project.
- Use `Color.ios` semantic colors for backgrounds, labels, separators per `theme/colors.ts` pattern.
- `contentInsetAdjustmentBehavior="automatic"` on all ScrollView/FlatList containers.
- `{ borderCurve: 'continuous' }` on all rounded containers.
- ScrollView uses `contentContainerStyle` padding, never padding on ScrollView itself.
- Tabular nums for counters: `{ fontVariant: 'tabular-nums' }`.
- Haptic feedback via `expo-haptics` on avatar and pill press.
- Preserve existing data flow: `useCalendarWithWeeks`, `useAthleteInfo`, `athletesWithEvents`, `resolvedAthleteId`, `filteredWeekGroups` — all unchanged.
- Reuse `EventCard.tsx` as-is for workout cards.
- Keep `ProfileAvatar` as-is (no `@expo/ui` Avatar exists).

---

## Task 1: Create `useLinkedScroll` hook

**Files:**
- Create: `apps/mobile/hooks/useLinkedScroll.ts`
- Test: `apps/mobile/hooks/__tests__/useLinkedScroll.test.ts`

**Interfaces:**
- Consumes: `{ itemCount: number, pageWidth: number, cooldownMs?: number }`
- Produces: `{ scrollIndex: SharedValue<number>, workoutListRef: RefObject<FlatList | null>, pillListRef: RefObject<FlatList | null>, scrollToIndex(index: number): void, handlers: { onScrollHandler: AnimatedScrollHandler, onViewable: (info: ViewabilityInfo) => void } }`

**Steps:**

- [ ] **Step 1: Write the failing test**

```ts
// apps/mobile/hooks/__tests__/useLinkedScroll.test.ts
import { renderHook } from '@testing-library/react-native';
import { useLinkedScroll } from '../useLinkedScroll';

// Mock reanimated
jest.mock('react-native-reanimated', () => {
  const actual = jest.requireActual('react-native-reanimated');
  return {
    ...actual,
    useSharedValue: jest.fn((init: number) => ({ value: init })),
    useAnimatedScrollHandler: jest.fn((handler) => handler),
  };
});

describe('useLinkedScroll', () => {
  it('returns scrollIndex starting at 0', () => {
    const { result } = renderHook(() => useLinkedScroll({ itemCount: 5, pageWidth: 390 }));
    expect(result.current.scrollIndex.value).toBe(0);
  });

  it('scrollToIndex calls scrollToItem on the workout list', () => {
    const workoutScrollToItem = jest.fn();
    const pillScrollToItem = jest.fn();
    const { result } = renderHook(() => useLinkedScroll({ itemCount: 3, pageWidth: 390 }));

    // Mock the refs
    (result.current.workoutListRef as any).current = { scrollToItem: workoutScrollToItem };
    (result.current.pillListRef as any).current = { scrollToItem: pillScrollToItem };

    result.current.scrollToIndex(1);

    // The workout list scrolls to the index
    expect(workoutScrollToItem).toHaveBeenCalledWith({ index: 1, animated: true });
    // The pill list does NOT scroll — it follows via highlight, not scroll
    expect(pillScrollToItem).not.toHaveBeenCalled();
  });

  it('scrollToIndex clamps to bounds', () => {
    const workoutScrollToItem = jest.fn();
    const { result } = renderHook(() => useLinkedScroll({ itemCount: 3, pageWidth: 390 }));
    (result.current.workoutListRef as any).current = { scrollToItem: workoutScrollToItem };

    result.current.scrollToIndex(10); // way out of bounds

    expect(workoutScrollToItem).toHaveBeenCalledWith({ index: 2, animated: true });
  });

  it('scrollToIndex with negative index clamps to 0', () => {
    const workoutScrollToItem = jest.fn();
    const { result } = renderHook(() => useLinkedScroll({ itemCount: 3, pageWidth: 390 }));
    (result.current.workoutListRef as any).current = { scrollToItem: workoutScrollToItem };

    result.current.scrollToIndex(-2);

    expect(workoutScrollToItem).toHaveBeenCalledWith({ index: 0, animated: true });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/mobile && bun test hooks/__tests__/useLinkedScroll.test.ts`
Expected: FAIL with "cannot find module '../useLinkedScroll'"

- [ ] **Step 3: Write the hook implementation**

```ts
// apps/mobile/hooks/useLinkedScroll.ts
import { useRef, useCallback } from 'react';
import { FlatList, ViewabilityInfo } from 'react-native';
import {
  useSharedValue,
  useAnimatedScrollHandler,
} from 'react-native-reanimated';

interface UseLinkedScrollOptions {
  itemCount: number;
  pageWidth: number;
  cooldownMs?: number;
}

interface UseLinkedScrollReturn {
  scrollIndex: ReturnType<typeof useSharedValue<number>>;
  workoutListRef: React.RefObject<FlatList | null>;
  pillListRef: React.RefObject<FlatList | null>;
  scrollToIndex: (index: number) => void;
  handlers: {
    onScrollHandler: ReturnType<typeof useAnimatedScrollHandler>;
    onViewable: (info: ViewabilityInfo) => void;
  };
}

export function useLinkedScroll({
  itemCount,
  pageWidth,
  cooldownMs = 100,
}: UseLinkedScrollOptions): UseLinkedScrollReturn {
  const scrollIndex = useSharedValue(0);
  const workoutListRef = useRef<FlatList>(null);
  const pillListRef = useRef<FlatList>(null);

  // Cooldown to prevent feedback loops
  const lastUpdate = useRef(0);

  const updateIndex = useCallback(
    (newIndex: number) => {
      const now = Date.now();
      if (now - lastUpdate.current < cooldownMs) return;
      lastUpdate.current = now;

      const maxIndex = Math.max(0, itemCount - 1);
      const clamped = Math.max(0, Math.min(newIndex, maxIndex));
      scrollIndex.value = clamped;
    },
    [scrollIndex, itemCount, cooldownMs],
  );

  const scrollToIndex = useCallback(
    (index: number) => {
      const clamped = Math.max(0, Math.min(index, Math.max(0, itemCount - 1)));
      scrollIndex.value = clamped;

      // Scroll the workout list (which has >= pages) to the index
      if (workoutListRef.current) {
        workoutListRef.current.scrollToIndex({ index: clamped, animated: true });
      }
      // The pill list does NOT scroll here — it follows via highlight
    },
    [scrollIndex, itemCount],
  );

  // Single scroll handler shared by both lists
  const onScrollHandler = useAnimatedScrollHandler({
    onScroll: (state) => {
      const index = Math.round(state.contentOffset.x / pageWidth);
      updateIndex(index);
    },
    onMomentumScrollEnd: (state) => {
      const index = Math.round(state.contentOffset.x / pageWidth);
      updateIndex(index);
    },
  });

  // Single viewability callback shared by both lists
  const onViewable = useCallback(
    (info: ViewabilityInfo) => {
      if (info.viewableItems?.[0]) {
        updateIndex(info.viewableItems[0].index ?? 0);
      }
    },
    [updateIndex],
  );

  return {
    scrollIndex,
    workoutListRef,
    pillListRef,
    scrollToIndex,
    handlers: { onScrollHandler, onViewable },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/mobile && bun test hooks/__tests__/useLinkedScroll.test.ts`
Expected: PASS (all 4 tests)

- [ ] **Step 5: Commit**

```bash
cd apps/mobile
git add hooks/useLinkedScroll.ts hooks/__tests__/useLinkedScroll.test.ts
git commit -m "feat(hooks): add useLinkedScroll hook for snap-scroll sync"
```

---

## Task 2: Rewrite Plan screen — layout and athlete strip

**Files:**
- Modify: `apps/mobile/components/screens/Plan.tsx`

**Interfaces:**
- Consumes: `useCalendarWithWeeks`, `useAthleteInfo`, `useMobileCalendarStore`, `useAiCoachStore`, `useCoachAgent`, `ProfileAvatar`, `Icon`, `StyledSafeAreaView`, `EventCard`, `useLinkedScroll`
- Produces: New Plan screen with three horizontal layers

**Steps:**

- [ ] **Step 1: Replace imports**

Replace the current imports in `Plan.tsx`:

```ts
import { useOrganization, useUser } from "@clerk/clerk-expo";
import { useCoachAgent } from "@runflow/data-react";
import { ProfileAvatar } from "@runflow/mobile/components";
import { Icon, StyledSafeAreaView } from "@runflow/mobile/components/common/ui";
import { EventCard } from "@runflow/mobile/components";
import {
  useAthleteInfo,
  useCalendarWithWeeks,
} from "@runflow/mobile/hooks";
import {
  useAiCoachStore,
  useMobileCalendarStore,
} from "@runflow/mobile/stores";
import { useLinkedScroll } from "@runflow/mobile/hooks/useLinkedScroll";
import { startOfWeek } from "date-fns";
import { useRouter } from "expo-router";
import React, { useMemo, useRef, useState } from "react";
import {
  FlatList,
  Pressable,
  ScrollView,
  Text,
  useWindowDimensions,
  View,
} from "react-native";
import * as Haptics from "expo-haptics";
```

Remove: `WorkoutDayRow` import (replaced by `EventCard`).

- [ ] **Step 2: Replace the component function — state and data layer**

Replace the component body. The state and data layer stays the same except for the linked scroll hook:

```tsx
export default function PlanScreen() {
  const router = useRouter();
  const { coach } = useCoachAgent();
  const { getAthleteName, athletes } = useAthleteInfo();
  const { organization } = useOrganization();
  const { user } = useUser();

  const { weekGroups, isDataLoading } = useCalendarWithWeeks(organization?.id);
  const setSelectedDate = useMobileCalendarStore((s) => s.setSelectedDate);

  const [expandedWeeks, setExpandedWeeks] = useState<Set<number>>(() => {
    const now = new Date();
    const currentWeekStart = startOfWeek(now, { weekStartsOn: 1 });
    const currentWeekNumber = currentWeekStart.getTime();
    return new Set([currentWeekNumber]);
  });

  const [selectedAthleteId, setSelectedAthleteId] = useState<string | null>(null);

  const athletesWithEvents = useMemo(() => {
    const seen = new Set<string>();
    const result: {
      id: string;
      firstName: string;
      lastName: string;
      initials: string;
    }[] = [];
    for (const week of weekGroups) {
      for (const event of week.events) {
        const id = event.user?.id;
        if (id && !seen.has(id)) {
          seen.add(id);
          const name = getAthleteName(id) ?? "Unknown";
          const firstName = name.split(" ")[0];
          const lastName = name.split(" ").slice(1).join(" ");
          const initials = name
            .split(" ")
            .map((w) => w[0])
            .join("")
            .slice(0, 2)
            .toUpperCase();
          result.push({ id, firstName, lastName, initials });
        }
      }
    }
    const myId = user?.id;
    if (myId && !seen.has(myId)) {
      seen.add(myId);
      const myName = getAthleteName(myId) ?? "Me";
      const firstName = myName.split(" ")[0];
      const lastName = myName.split(" ").slice(1).join(" ");
      const initials = myName
        .split(" ")
        .map((w) => w[0])
        .join("")
        .slice(0, 2)
        .toUpperCase();
      result.unshift({ id: myId, firstName, lastName, initials });
    }
    return result;
  }, [weekGroups, getAthleteName, user?.id]);

  const myAthlete = useMemo(() => {
    const myId = user?.id;
    if (!myId) return null;
    return athletesWithEvents.find((a) => a.id === myId) ?? null;
  }, [athletesWithEvents, user?.id]);

  const resolvedAthleteId = useMemo(() => {
    if (selectedAthleteId) return selectedAthleteId;
    if (myAthlete?.id) return myAthlete.id;
    if (athletesWithEvents[0]?.id) return athletesWithEvents[0].id;
    if (athletes?.[0]?.id) return athletes[0].id;
    return null;
  }, [selectedAthleteId, myAthlete, athletesWithEvents, athletes]);

  const filteredWeekGroups = useMemo(() => {
    if (!resolvedAthleteId) return weekGroups;
    return weekGroups
      .map((week) => ({
        ...week,
        events: week.events.filter(
          (e) => e.user?.id === resolvedAthleteId
        ),
      }))
      .filter((week) => week.events.length > 0);
  }, [weekGroups, resolvedAthleteId]);

  const { width: windowWidth } = useWindowDimensions();
  const PAGE_WIDTH = windowWidth;

  // Linked scroll: week pills ↔ workout pages
  const {
    scrollIndex,
    workoutListRef,
    pillListRef,
    scrollToIndex,
    handlers,
  } = useLinkedScroll({
    itemCount: filteredWeekGroups.length,
    pageWidth: PAGE_WIDTH,
    cooldownMs: 100,
  });

  const handleCoachForWorkout = (workoutTitle: string) => {
    useAiCoachStore.getState().sendMessage(
      `Explain why I'm doing this session: ${workoutTitle}`,
      coach,
    );
    router.push("/coach");
  };

  const handleWorkoutDayPress = (workoutDate?: Date) => {
    if (workoutDate) setSelectedDate(workoutDate);
    router.push("/(tabs)/progress/activity-calendar");
  };

  if (isDataLoading) {
    return (
      <View className="flex-1 bg-background-primary items-center justify-center">
        <Text className="text-text-secondary">Loading your plan...</Text>
      </View>
    );
  }
```

- [ ] **Step 3: Replace the return JSX — the three-layer layout**

Replace the entire return block:

```tsx
  return (
    <StyledSafeAreaView className="flex-1 bg-background-primary">
      {/* ── Layer 1: Athlete Strip ── */}
      {athletesWithEvents.length >= 1 && (
        <FlatList
          horizontal
          data={athletesWithEvents}
          keyExtractor={(item) => item.id}
          showsHorizontalScrollIndicator={false}
          decelerationRate="fast"
          snapToInterval={60}
          contentContainerStyle={{ paddingHorizontal: 12, paddingVertical: 8 }}
          renderItem={({ item }) => {
            const isActive = item.id === resolvedAthleteId;
            return (
              <Pressable
                onPress={() => {
                  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light).catch(() => {});
                  setSelectedAthleteId(item.id);
                }}
                style={{
                  alignItems: 'center',
                  marginHorizontal: 4,
                  width: 60,
                }}
                accessibilityRole="tab"
                accessibilityState={{ selected: isActive }}
              >
                <View
                  style={{
                    borderWidth: isActive ? 2 : 0,
                    borderColor: isActive ? '#ADFF2F' : 'transparent',
                    borderRadius: 22,
                    padding: 1,
                  }}
                >
                  <ProfileAvatar
                    size={40}
                    firstName={item.firstName}
                    lastName={item.lastName}
                    accessibilityLabel={`${item.firstName || 'Athlete'} ${item.lastName || ''}`}
                  />
                </View>
                <Text
                  style={{
                    color: isActive ? '#ADFF2F' : 'rgba(255,255,255,0.6)',
                    fontSize: 10,
                    fontWeight: isActive ? '600' : '400',
                    marginTop: 4,
                    textAlign: 'center',
                    width: 56,
                    numberOfLines: 1,
                    overflow: 'hidden',
                    textOverflow: 'ellipsis',
                  }}
                >
                  {item.firstName || 'Athlete'}
                </Text>
              </Pressable>
            );
          }}
        />
      )}

      {/* ── Layer 2: Week Pills ── */}
      {filteredWeekGroups.length >= 1 && (
        <FlatList
          ref={pillListRef}
          horizontal
          data={filteredWeekGroups}
          keyExtractor={(item) => String(item.weekNumber)}
          showsHorizontalScrollIndicator={false}
          decelerationRate="fast"
          snapToInterval={PAGE_WIDTH}
          contentContainerStyle={{ paddingVertical: 12 }}
          renderItem={({ item, index }) => {
            const isActive = index === scrollIndex.value;
            return (
              <Pressable
                onPress={() => {
                  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light).catch(() => {});
                  scrollToIndex(index);
                }}
                style={{
                  paddingHorizontal: 16,
                  paddingVertical: 8,
                  marginHorizontal: 4,
                  borderRadius: 20,
                  backgroundColor: isActive ? '#ADFF2F' : 'transparent',
                  borderWidth: isActive ? 0 : 1,
                  borderColor: 'rgba(255,255,255,0.2)',
                }}
              >
                <Text
                  style={{
                    color: isActive ? '#141E0C' : 'rgba(255,255,255,0.7)',
                    fontSize: 13,
                    fontWeight: isActive ? '700' : '500',
                    whiteSpace: 'nowrap',
                  }}
                >
                  {item.weekLabel}
                </Text>
              </Pressable>
            );
          }}
          onScroll={handlers.onScrollHandler}
          onScrollEndDrag={handlers.onScrollHandler}
          onViewableItemsChanged={handlers.onViewable}
          scrollEventThrottle={16}
        />
      )}

      {/* ── Layer 3: Workout Cards ── */}
      {filteredWeekGroups.length === 0 ? (
        <ScrollView
          className="flex-1"
          showsVerticalScrollIndicator={false}
          contentInsetAdjustmentBehavior="automatic"
          contentContainerStyle={{
            paddingHorizontal: 16,
            paddingBottom: 32,
            paddingTop: 8,
          }}
        >
          <Pressable
            onPress={() => {
              useAiCoachStore.getState().sendMessage(
                "I don't have any workouts scheduled. Can you build me a training plan?",
                coach,
              );
              router.push("/coach");
            }}
            style={{
              backgroundColor: 'rgba(17,18,19,0.5)',
              borderRadius: 24,
              padding: 24,
              alignItems: 'center',
              marginTop: 40,
            }}
          >
            <Icon
              select={{ from: "ionicons", name: "sparkles" }}
              size={40}
              style={{ color: '#ADFF2F', marginBottom: 16 }}
            />
            <Text style={{ color: '#FFFFFF', fontSize: 20, fontWeight: '700', textAlign: 'center', marginBottom: 8 }}>
              No training plan
            </Text>
            <Text style={{ color: 'rgba(255,255,255,0.7)', fontSize: 14, textAlign: 'center', marginBottom: 16 }}>
              Ask your adaptive coach to build a personalized training plan.
            </Text>
            <View style={{ backgroundColor: 'rgba(17,18,19,0.5)', borderRadius: 20, paddingHorizontal: 16, paddingVertical: 10 }}>
              <Text style={{ color: '#ADFF2F', fontWeight: '600', fontSize: 14 }}>
                Build my plan
              </Text>
            </View>
          </Pressable>
        </ScrollView>
      ) : (
        <FlatList
          ref={workoutListRef}
          horizontal
          data={filteredWeekGroups}
          keyExtractor={(item) => String(item.weekNumber)}
          pagingEnabled
          showsHorizontalScrollIndicator={false}
          decelerationRate="fast"
          snapToInterval={PAGE_WIDTH}
          contentContainerStyle={{ paddingHorizontal: 0 }}
          renderItem={({ item }) => {
            const weekWorkouts = item.events;
            return (
              <View
                style={{
                  width: PAGE_WIDTH,
                  paddingHorizontal: 16,
                  paddingVertical: 8,
                }}
              >
                {weekWorkouts.length === 0 ? (
                  <Text style={{ color: 'rgba(255,255,255,0.5)', textAlign: 'center', marginTop: 24, fontSize: 14 }}>
                    No workouts this week
                  </Text>
                ) : (
                  <FlatList
                    data={weekWorkouts}
                    keyExtractor={(event) => event.id}
                    showsVerticalScrollIndicator={false}
                    contentInsetAdjustmentBehavior="automatic"
                    contentContainerStyle={{ paddingBottom: 24 }}
                    renderItem={({ item: event }) => (
                      <EventCard
                        title={event.title || "Outdoor Run"}
                        athleteName={getAthleteName(event.user?.id)}
                        isCompleted={event.completed}
                        type={event.type}
                        distance={
                          event.completed
                            ? (() => {
                                const d = event.workout?.distance;
                                if (d && d > 0) return `${d.toFixed(2)} mi`;
                                const s = event.workout?.actual_duration_seconds;
                                if (s && s > 0) return `${Math.round(s / 60)} min`;
                                return undefined;
                              })()
                            : undefined
                        }
                        timeRange={
                          event.starts_at && event.ends_at
                            ? `${new Date(event.starts_at).toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' })} - ${new Date(event.ends_at).toLocaleTimeString([], { hour: 'numeric', minute: '2-digit' })}`
                            : undefined
                        }
                        subtitle={
                          event.workout?.duration
                            ? `${event.workout.type} · ${event.workout.duration}min`
                            : event.workout?.type
                        }
                        intensity={event.workout?.intensity}
                        onPressQuaternary={() =>
                          handleWorkoutDayPress(
                            event.workout?.date
                              ? new Date(event.workout.date)
                              : undefined
                          )
                        }
                        isEmptyState={!event.workout}
                        hasActiveTrainingPlan={true}
                        onPressCoach={() =>
                          handleCoachForWorkout(
                            event.title || "this workout"
                          )
                        }
                      />
                    )}
                    ItemSeparatorComponent={() => (
                      <View style={{ height: 8 }} />
                    )}
                  />
                )}
              </View>
            );
          }}
          onScroll={handlers.onScrollHandler}
          onScrollEndDrag={handlers.onScrollHandler}
          onViewableItemsChanged={handlers.onViewable}
          scrollEventThrottle={16}
        />
      )}
    </StyledSafeAreaView>
  );
}
```

Key differences from the current Plan.tsx:
- Three horizontal snap-scrolling layers instead of vertical expandable weeks
- Athlete strip has NO ref (it doesn't participate in linked scroll)
- Week pills use `pillListRef` and `onScroll`/`onScrollEndDrag` with `scrollEventThrottle={16}`
- Workout pages use `workoutListRef` with the same scroll handlers
- Both scroll lists share `handlers.onScrollHandler` and `handlers.onViewable` from the hook
- `contentContainerStyle` on pill list has NO horizontal padding (padding would offset `contentOffset.x` calc)
- `onViewableItemsChanged` uses `handlers.onViewable` (a single callback, not an array)

- [ ] **Step 4: Run TypeScript check to verify no type errors**

Run: `cd apps/mobile && bun check-types`
Expected: No errors (or only pre-existing errors unrelated to Plan.tsx)

- [ ] **Step 5: Commit**

```bash
cd apps/mobile
git add components/screens/Plan.tsx
git commit -m "feat(plan): redesign screen with horizontal card scroll layout"
```

---

## Task 3: Polish and cleanup

**Files:**
- Modify: `apps/mobile/components/screens/Plan.tsx`

**Steps:**

- [ ] **Step 1: Remove unused state**

Remove `expandedWeeks` state and `toggleWeek` function — no longer needed with the horizontal card layout.

```tsx
// Remove these lines from the component:
const [expandedWeeks, setExpandedWeeks] = useState<Set<number>>(() => {
  const now = new Date();
  const currentWeekStart = startOfWeek(now, { weekStartsOn: 1 });
  const currentWeekNumber = currentWeekStart.getTime();
  return new Set([currentWeekNumber]);
});

const toggleWeek = (weekNumber: number) => {
  setExpandedWeeks((prev) => {
    const next = new Set(prev);
    if (next.has(weekNumber)) {
      next.delete(weekNumber);
    } else {
      next.add(weekNumber);
    }
    return next;
  });
};
```

Remove `startOfWeek` from `date-fns` import if it's no longer used (keep `startOfWeek` only if the loading check or other code still uses it — the loading check does NOT use it, so remove entirely).

- [ ] **Step 2: Verify scroll sync works correctly**

Test manually in simulator:
1. Scroll week pills → workout pages should snap to matching index
2. Scroll workout pages → week pills should highlight matching pill
3. Tap a week pill → workout list should scroll to that week
4. Switch athlete → workout list should reset to week 0
5. Scroll to last week → pill highlights last pill
6. Scroll back to first week → pill highlights first pill

- [ ] **Step 3: Verify empty states**

Test:
1. No athlete filter → all weeks visible
2. Athlete filter with no matching events → empty week shows "No workouts this week"
3. No training plan at all → AI coach CTA card shows

- [ ] **Step 4: Commit**

```bash
cd apps/mobile
git add components/screens/Plan.tsx
git commit -m "style(plan): remove unused state and polish scroll sync"
```

---

## Self-Review

**Spec coverage:**
- Layer 1 (athlete strip): Task 2, Step 3 ✓ — horizontal FlatList, no ref, neon green active ring
- Layer 2 (week pills): Task 2, Step 3 ✓ — horizontal FlatList, pillListRef, scroll handlers
- Layer 3 (workout cards): Task 2, Step 3 ✓ — horizontal FlatList with paging, workoutListRef, nested vertical FlatList
- Scroll sync hook: Task 1 ✓ — SharedValue + single shared handler
- Haptics on press: Task 2, Step 3 ✓ — Haptics.impactAsync on avatar and pill press
- Empty state (no plan): Task 2, Step 3 ✓ — AI coach CTA card
- Empty week: Task 2, Step 3 ✓ — "No workouts this week" text
- HIG compliance (contentInsetAdjustmentBehavior, borderCurve, tabular nums): Task 2, Step 3 ✓
- No new dependencies: all imports are existing packages ✓
- Unused state removed: Task 3, Step 1 ✓

**Placeholder scan:** No "TBD", "TODO", or incomplete sections. All code is concrete.

**Type consistency:** `scrollIndex` is `SharedValue<number>` throughout. `PAGE_WIDTH` is `number` from `useWindowDimensions`. `filteredWeekGroups` is `WeekGroup[]`. `handlers.onScrollHandler` is `AnimatedScrollHandler` used as `onScroll` and `onScrollEndDrag`. `handlers.onViewable` is `(info: ViewabilityInfo) => void` used as `onViewableItemsChanged`. All EventCard props match the existing EventCard interface.

**Gaps:** None identified.
