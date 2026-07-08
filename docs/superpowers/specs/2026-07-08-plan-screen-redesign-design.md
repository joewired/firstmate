# Plan Screen Redesign — Design Spec

**Date:** 2026-07-08
**Project:** Run Flow (mobile)
**File:** `apps/mobile/components/screens/Plan.tsx`

---

## 1. Overview

Redesign the Plan screen from a vertical expandable week list into a horizontal card-scroll layout inspired by Runna and Kotcha. Three horizontal layers stacked vertically: athlete avatars, week pills, and workout cards. The week pills and workout cards are linked — scrolling one scrolls the other.

---

## 2. Layout Structure

Three horizontal layers, each a horizontal snap-scrolling list:

| Layer | Content | Scroll | Interaction |
|-------|---------|--------|-------------|
| 1 | Athlete avatars | Horizontal snap | Tap filters workouts below |
| 2 | Week pills ("Week 1", "Week 2"...) | Horizontal snap | Tap scrolls workout list to that week |
| 3 | Workout cards per week | Horizontal snap (pages), vertical scroll within each page | Tap opens activity calendar |

Each layer in Layer 3 is a full-width page containing a vertical list of `EventCard` components.

---

## 3. Layer 1: Athlete Strip

- **Component:** Existing `ProfileAvatar` component (unchanged — `@expo/ui` does not ship an `Avatar` component).
- **Layout:** Horizontal `FlatList` with `snapToInterval`, `decelerationRate: 'fast'`, `showsHorizontalScrollIndicator={false}`.
- **Active state:** Neon green ring (`NEON_GREEN`) around the avatar, achieved with a wrapper `View` that has `borderWidth: 2, borderColor: NEON_GREEN, borderRadius: size/2`.
- **Data:** Same `athletesWithEvents` logic — derives from week group events + current user, with `resolvedAthleteId` for preselection.
- **Interaction:** Tap sets `selectedAthleteId`, which filters Layer 3 workouts. Haptic feedback via `expo-haptics` on press.

---

## 4. Layer 2: Week Pills

- **Layout:** Horizontal `FlatList` with snap scrolling, pill-style text chips.
- **Label format:** `WEEK N • date range` (e.g., `WEEK 2 • Jul 13–19`), from `weekLabel` in `WeekGroup`.
- **Active pill:** Bold text, `NEON_GREEN` background, dark text color.
- **Inactive pill:** Outlined/ghost style — light border, `text-text-secondary`.
- **Interaction:** Tap calls `scrollToIndex` on the workout list ref to jump to that week's page.
- **Scroll sync:** Uses Reanimated `SharedValue<number>` for `contentOffset.x`. Both this list and the workout list read the shared value and adjust their offset accordingly.

---

## 5. Layer 3: Workout Cards

- **Layout:** Horizontal `FlatList` with `pagingEnabled: true`, `snapToInterval`, `decelerationRate: 'fast'`.
- **Each page:** One week, rendered as a full-width `View` containing a vertical `FlatList` of `EventCard` components.
- **Nested scroll:** Each page's vertical list scrolls independently. React Native's `nestedScrollEnabled` handles the nesting.
- **Card component:** Reuses `EventCard.tsx` directly — same dark card (`NEON_GREEN_DARKER`), same icon/title/completion/actions layout.
- **Empty weeks:** Skipped via the existing `filteredWeekGroups` filter (weeks with zero events after athlete filter are dropped).
- **Empty state:** If no workouts exist at all, show the AI coach CTA card (sparkles icon, "No training plan" / "Ask Coach" button) — same as current.

---

## 6. Scroll Sync Mechanism

Both Layer 2 (week pills) and Layer 3 (workout pages) share a Reanimated `SharedValue<number>` for scroll index:

```ts
const scrollIndex = useSharedValue(0);
const PAGE_WIDTH = windowWidth; // from useWindowDimensions
```

**Sync logic:**

1. `useAnimatedScrollHandler` on both lists reads `contentOffset.x` and updates `scrollIndex.value = Math.round(x / PAGE_WIDTH)`.
2. When `scrollIndex.value` changes, both lists call `scrollToIndex` on their respective refs to stay in sync.
3. Debounce with `Animated.add` and `Animated.timing` to prevent feedback loops (set a 100ms cooldown so scrolling one doesn't trigger the other's handler in a loop).
4. `onViewableItemsChanged` on both lists for instant index sync when the user is actively scrolling.

---

## 7. Styling & HIG Compliance

- **Colors:** `Color.ios` semantic colors for backgrounds, labels, separators (as defined in `theme/colors.ts` pattern).
- **Safe area:** `contentInsetAdjustmentBehavior="automatic"` on the root ScrollView.
- **Corners:** `{ borderCurve: 'continuous' }` on all rounded containers.
- **Animations:** Entering/exiting animations on cards via `@expo/ui/swift-ui` `Animation` modifiers.
- **Padding:** ScrollView uses `contentContainerStyle` padding, not padding on ScrollView itself.
- **Typography:** Tabular nums for any counters (`fontVariant: 'tabular-nums'`).
- **Haptics:** `expo-haptics` on avatar and pill press.

---

## 8. Data Flow

```
useCalendarWithWeeks(organizationId)
  → weekGroups: WeekGroup[]
  → filteredWeekGroups: WeekGroup[] (filtered by selectedAthleteId)

athletesWithEvents (useMemo)
  → derived from weekGroups.events + current user
  → resolvedAthleteId (preselected)

EventCard props (per workout):
  title: event.title || "Outdoor Run"
  athleteName: getAthleteName(event.user?.id)
  isCompleted: event.completed
  type: event.type
  distance: completedMetric (distance or time)
  timeRange: starts_at – ends_at
  subtitle: type • duration
  intensity: event.workout?.intensity
  onPressQuaternary: router.push("/today/workout")
```

Same data flow as current Plan.tsx — no changes to hooks or stores.

---

## 9. Dependencies

- `@expo/ui` — `Host`, `Column`, `Row`, `Icon`, `Text` (already installed, SDK 57)
- `react-native-reanimated` — `useSharedValue`, `useAnimatedScrollHandler`, `Animated.ScrollView`, `Animated.add`, `Animated.timing` (already in use via `@expo/ui/swift-ui` animation modifiers)
- `expo-haptics` — press feedback (already installed)
- `expo-router` — `Color`, `useRouter` (already installed)
- `expo-image` — avatar image loading (already installed)

No new dependencies required.

---

## 10. Files Changed

| File | Change |
|------|--------|
| `apps/mobile/components/screens/Plan.tsx` | Full redesign — new layout, scroll sync, athlete strip, week pills |
| `apps/mobile/hooks/today/useCalendarWithWeeks.ts` | No change (data provider, unchanged) |
| `apps/mobile/components/calendar/EventCard.tsx` | No change (reused as-is) |
| `apps/mobile/constants/colors.ts` | No change (existing colors reused) |

---

## 11. Rejected Alternatives

- **Vertical expandable weeks (current):** Rejected — less modern, less thumb-friendly, doesn't match Runna/Kotcha quality bar.
- **Third-party linked-scroll library:** Rejected — Reanimated SharedValue sync is more performant, the app already uses Reanimated, and a third-party lib adds dependency surface without clear benefit.
- **Full calendar grid:** Rejected — captain explicitly wants a simplified layout, no calendar grid needed.
