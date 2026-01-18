import React, { useEffect, useMemo, useState } from 'react';
import {
  Alert,
  Keyboard,
  SafeAreaView,
  StyleSheet,
  Text,
  TouchableOpacity,
  View,
} from 'react-native';
import { FlashList } from '@shopify/flash-list';
import * as Haptics from 'expo-haptics';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { router } from 'expo-router';

import { ExpenseFormModal } from '../src/components/ExpenseFormModal';
import { ExpenseRow } from '../src/components/ExpenseRow';
import { useExpensesStore } from '../src/store/expensesStore';
import { formatCurrencyFromCents } from '../src/utils/money';

function startOfToday() {
  const d = new Date();
  d.setHours(0, 0, 0, 0);
  return d.getTime();
}

function Separator() {
  return <View style={{ height: 10 }} />;
}

export default function Index() {
  const insets = useSafeAreaInsets();
  const { hydrated, expenses, error, hydrate, clearError, addExpense, updateExpense, deleteExpense } =
    useExpensesStore();

  const [modalVisible, setModalVisible] = useState(false);
  const [editing, setEditing] = useState(null);
  const [busy, setBusy] = useState(false);

  useEffect(() => {
    hydrate();
  }, [hydrate]);

  const totals = useMemo(() => {
    const todayStart = startOfToday();
    let todayCents = 0;
    let allCents = 0;
    for (const e of expenses) {
      allCents += e.amountCents;
      const t = new Date(e.createdAt).getTime();
      if (t >= todayStart) todayCents += e.amountCents;
    }
    return { todayCents, allCents };
  }, [expenses]);

  const openAdd = () => {
    Keyboard.dismiss();
    setEditing(null);
    setModalVisible(true);
  };

  const openEdit = (expense) => {
    Keyboard.dismiss();
    setEditing(expense);
    setModalVisible(true);
  };

  const closeModal = () => {
    setModalVisible(false);
    setEditing(null);
  };

  const handleDelete = (expense) => {
    Alert.alert('Delete expense', 'This cannot be undone.', [
      { text: 'Cancel', style: 'cancel' },
      {
        text: 'Delete',
        style: 'destructive',
        onPress: async () => {
          try {
            await Haptics.notificationAsync(Haptics.NotificationFeedbackType.Warning);
          } catch {
            // ignore
          }
          await deleteExpense(expense.id);
        },
      },
    ]);
  };

  const submit = async (values) => {
    setBusy(true);
    try {
      if (editing) {
        await updateExpense({
          ...editing,
          amountCents: values.amountCents,
          category: values.category,
          note: values.note,
        });
      } else {
        const id = `${Date.now()}_${Math.random().toString(16).slice(2)}`;
        await addExpense({
          id,
          amountCents: values.amountCents,
          category: values.category,
          note: values.note,
          createdAt: new Date().toISOString(),
        });
      }

      try {
        await Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
      } catch {
        // ignore
      }

      closeModal();
    } finally {
      setBusy(false);
    }
  };

  return (
    <SafeAreaView style={[styles.container, { paddingTop: Math.max(insets.top, 12) }]}>
      <View style={styles.header}>
        <View style={{ flex: 1 }}>
          <Text style={styles.title}>Expenses</Text>
          <Text style={styles.subtitle}>Simple local tracker</Text>
        </View>

        <TouchableOpacity
          onPress={() => router.push('/monthly')}
          accessibilityRole="button"
          style={styles.headerBtn}
        >
          <Text style={styles.headerBtnText}>Monthly</Text>
        </TouchableOpacity>

        <TouchableOpacity onPress={openAdd} accessibilityRole="button" style={styles.headerBtn}>
          <Text style={styles.headerBtnText}>Add</Text>
        </TouchableOpacity>
      </View>

      {error ? (
        <TouchableOpacity
          accessibilityRole="button"
          onPress={clearError}
          style={styles.errorBanner}
          activeOpacity={0.85}
        >
          <Text style={styles.errorBannerText}>{error} Tap to dismiss.</Text>
        </TouchableOpacity>
      ) : null}

      <View style={styles.summaryRow}>
        <View style={styles.card}>
          <Text style={styles.cardLabel}>Today</Text>
          <Text style={styles.cardValue}>{formatCurrencyFromCents(totals.todayCents)}</Text>
        </View>
        <View style={styles.card}>
          <Text style={styles.cardLabel}>All time</Text>
          <Text style={styles.cardValue}>{formatCurrencyFromCents(totals.allCents)}</Text>
        </View>
      </View>

      {!hydrated ? (
        <View style={styles.center}>
          <Text style={styles.loadingText}>Loading…</Text>
        </View>
      ) : expenses.length === 0 ? (
        <View style={styles.center}>
          <Text style={styles.emptyTitle}>No expenses yet</Text>
          <Text style={styles.emptySub}>Tap “Add” to record your first expense.</Text>
        </View>
      ) : (
        <FlashList
          data={expenses}
          keyExtractor={(item) => item.id}
          estimatedItemSize={76}
          contentContainerStyle={{ paddingHorizontal: 16, paddingBottom: 16 }}
          ItemSeparatorComponent={Separator}
          renderItem={({ item }) => (
            <ExpenseRow
              expense={item}
              onPress={() => openEdit(item)}
              onLongPress={() => handleDelete(item)}
            />
          )}
        />
      )}

      <ExpenseFormModal
        visible={modalVisible}
        mode={editing ? 'edit' : 'add'}
        initial={editing}
        onClose={closeModal}
        onSubmit={submit}
        busy={busy}
      />

      <View style={[styles.bottomBar, { paddingBottom: Math.max(insets.bottom, 16) }]}>
        <TouchableOpacity onPress={openAdd} accessibilityRole="button" style={styles.primaryCta}>
          <Text style={styles.primaryCtaText}>Add expense</Text>
        </TouchableOpacity>
      </View>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#0B0B0B' },
  header: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingHorizontal: 16,
    paddingVertical: 12,
    gap: 12,
  },
  title: { color: '#F2F2F2', fontSize: 26, fontWeight: '900', letterSpacing: -0.3 },
  subtitle: { marginTop: 2, color: '#9A9A9A', fontSize: 13, fontWeight: '600' },
  headerBtn: {
    minHeight: 44,
    paddingHorizontal: 14,
    borderRadius: 14,
    backgroundColor: 'rgba(255,255,255,0.10)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  headerBtnText: { color: '#F0F0F0', fontWeight: '800', fontSize: 14 },
  errorBanner: {
    marginHorizontal: 16,
    marginBottom: 12,
    paddingHorizontal: 14,
    paddingVertical: 12,
    borderRadius: 16,
    backgroundColor: 'rgba(255, 90, 90, 0.18)',
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255, 90, 90, 0.35)',
  },
  errorBannerText: { color: '#FFB4B4', fontSize: 12, fontWeight: '700' },
  summaryRow: { flexDirection: 'row', gap: 12, paddingHorizontal: 16, paddingBottom: 12 },
  card: {
    flex: 1,
    borderRadius: 18,
    padding: 14,
    backgroundColor: '#121212',
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255,255,255,0.08)',
  },
  cardLabel: { color: '#A7A7A7', fontSize: 12, fontWeight: '700' },
  cardValue: { marginTop: 10, color: '#F2F2F2', fontSize: 18, fontWeight: '900' },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center', paddingHorizontal: 24 },
  loadingText: { color: '#CFCFCF', fontSize: 14, fontWeight: '700' },
  emptyTitle: { color: '#F2F2F2', fontSize: 18, fontWeight: '900' },
  emptySub: {
    marginTop: 8,
    textAlign: 'center',
    color: '#9A9A9A',
    fontSize: 13,
    fontWeight: '600',
    lineHeight: 18,
  },
  bottomBar: {
    paddingHorizontal: 16,
    paddingTop: 10,
    backgroundColor: 'rgba(11,11,11,0.96)',
    borderTopWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255,255,255,0.08)',
  },
  primaryCta: {
    minHeight: 52,
    borderRadius: 18,
    backgroundColor: '#E8E8E8',
    alignItems: 'center',
    justifyContent: 'center',
  },
  primaryCtaText: { color: '#0B0B0B', fontSize: 16, fontWeight: '900' },
});
2) Monthly Summary screen
/app/frontend/app/monthly.tsx

import React, { useMemo, useState } from 'react';
import { SafeAreaView, StyleSheet, Text, TouchableOpacity, View } from 'react-native';
import { useSafeAreaInsets } from 'react-native-safe-area-context';
import { addMonths, format, startOfMonth } from 'date-fns';
import { router } from 'expo-router';

import { useExpensesStore } from '../src/store/expensesStore';
import { formatCurrencyFromCents } from '../src/utils/money';

const CATEGORIES = ['Food', 'Transport', 'Bills', 'Shopping', 'Other'];

function monthKey(d) {
  return format(d, 'yyyy-MM');
}

export default function Monthly() {
  const insets = useSafeAreaInsets();
  const expenses = useExpensesStore((s) => s.expenses);

  const [cursor, setCursor] = useState(() => startOfMonth(new Date()));

  const summary = useMemo(() => {
    const start = startOfMonth(cursor);
    const end = startOfMonth(addMonths(start, 1));

    const byCategory = {
      Food: 0,
      Transport: 0,
      Bills: 0,
      Shopping: 0,
      Other: 0,
    };

    let total = 0;

    for (const e of expenses) {
      const t = new Date(e.createdAt);
      if (t >= start && t < end) {
        total += e.amountCents;
        byCategory[e.category] = (byCategory[e.category] ?? 0) + e.amountCents;
      }
    }

    const rows = CATEGORIES.map((c) => ({ category: c, amountCents: byCategory[c] ?? 0 }))
      .filter((r) => r.amountCents > 0)
      .sort((a, b) => b.amountCents - a.amountCents);

    return { start, total, rows, empty: total === 0 };
  }, [cursor, expenses]);

  const title = useMemo(() => format(cursor, 'MMMM yyyy'), [cursor]);

  return (
    <SafeAreaView style={[styles.container, { paddingTop: Math.max(insets.top, 12) }]}>
      <View style={styles.header}>
        <TouchableOpacity accessibilityRole="button" onPress={() => router.back()} style={styles.backBtn}>
          <Text style={styles.backBtnText}>Back</Text>
        </TouchableOpacity>

        <View style={{ flex: 1 }}>
          <Text style={styles.title}>Monthly summary</Text>
          <Text style={styles.subtitle}>{title}</Text>
        </View>
      </View>

      <View style={styles.navRow}>
        <TouchableOpacity accessibilityRole="button" onPress={() => setCursor((d) => addMonths(d, -1))} style={styles.navBtn}>
          <Text style={styles.navBtnText}>Previous</Text>
        </TouchableOpacity>

        <View style={styles.monthPill}>
          <Text style={styles.monthPillText}>{monthKey(summary.start)}</Text>
        </View>

        <TouchableOpacity accessibilityRole="button" onPress={() => setCursor((d) => addMonths(d, 1))} style={styles.navBtn}>
          <Text style={styles.navBtnText}>Next</Text>
        </TouchableOpacity>
      </View>

      <View style={styles.totalCard}>
        <Text style={styles.totalLabel}>Total</Text>
        <Text style={styles.totalValue}>{formatCurrencyFromCents(summary.total)}</Text>
      </View>

      {summary.empty ? (
        <View style={styles.center}>
          <Text style={styles.emptyTitle}>No expenses this month</Text>
          <Text style={styles.emptySub}>Switch months or add an expense.</Text>
        </View>
      ) : (
        <View style={styles.listWrap}>
          {summary.rows.map((r) => (
            <View key={r.category} style={styles.row}>
              <Text style={styles.rowLabel}>{r.category}</Text>
              <Text style={styles.rowValue}>{formatCurrencyFromCents(r.amountCents)}</Text>
            </View>
          ))}
        </View>
      )}
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#0B0B0B', paddingHorizontal: 16 },
  header: { paddingVertical: 12, flexDirection: 'row', alignItems: 'center', gap: 12 },
  backBtn: {
    minHeight: 44,
    paddingHorizontal: 12,
    borderRadius: 14,
    backgroundColor: 'rgba(255,255,255,0.10)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  backBtnText: { color: '#F0F0F0', fontWeight: '800', fontSize: 13 },
  title: { color: '#F2F2F2', fontSize: 24, fontWeight: '900', letterSpacing: -0.3 },
  subtitle: { marginTop: 4, color: '#9A9A9A', fontSize: 13, fontWeight: '700' },

  navRow: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', gap: 10, marginTop: 6 },
  navBtn: {
    minHeight: 44,
    paddingHorizontal: 12,
    borderRadius: 14,
    backgroundColor: 'rgba(255,255,255,0.10)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  navBtnText: { color: '#F0F0F0', fontWeight: '800', fontSize: 13 },
  monthPill: {
    flex: 1,
    minHeight: 44,
    borderRadius: 14,
    backgroundColor: '#121212',
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255,255,255,0.08)',
    alignItems: 'center',
    justifyContent: 'center',
  },
  monthPillText: { color: '#CFCFCF', fontSize: 12, fontWeight: '800' },

  totalCard: {
    marginTop: 14,
    borderRadius: 18,
    padding: 14,
    backgroundColor: '#121212',
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255,255,255,0.08)',
  },
  totalLabel: { color: '#A7A7A7', fontSize: 12, fontWeight: '800' },
  totalValue: { marginTop: 10, color: '#F2F2F2', fontSize: 22, fontWeight: '900' },

  listWrap: { marginTop: 14, gap: 10 },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingVertical: 14,
    paddingHorizontal: 14,
    borderRadius: 16,
    backgroundColor: '#141414',
    borderWidth: StyleSheet.hairlineWidth,
    borderColor: 'rgba(255,255,255,0.08)',
  },
  rowLabel: { color: '#F2F2F2', fontSize: 14, fontWeight: '900' },
  rowValue: { color: '#F2F2F2', fontSize: 14, fontWeight: '900' },

  center: { flex: 1, alignItems: 'center', justifyContent: 'center', paddingHorizontal: 24 },
  emptyTitle: { color: '#F2F2F2', fontSize: 18, fontWeight: '900' },
  emptySub: { marginTop: 8, textAlign: 'center', color: '#9A9A9A', fontSize: 13, fontWeight: '600', lineHeight: 18 },
});
3) Local storage + state (Zustand + AsyncStorage)
/app/frontend/src/store/expensesStore.ts

import AsyncStorage from '@react-native-async-storage/async-storage';
import { create } from 'zustand';
import { Expense } from '../types/expense';

const STORAGE_KEY = 'expenses.v1';

type ExpensesState = {
  hydrated: boolean;
  expenses: Expense[];
  error?: string;
  hydrate: () => Promise<void>;
  addExpense: (expense: Expense) => Promise<void>;
  updateExpense: (expense: Expense) => Promise<void>;
  deleteExpense: (id: string) => Promise<void>;
  clearError: () => void;
};

async function persist(expenses: Expense[]) {
  await AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(expenses));
}

export const useExpensesStore = create<ExpensesState>((set, get) => ({
  hydrated: false,
  expenses: [],
  error: undefined,

  clearError: () => set({ error: undefined }),

  hydrate: async () => {
    try {
      const raw = await AsyncStorage.getItem(STORAGE_KEY);
      const parsed = raw ? (JSON.parse(raw) as Expense[]) : [];
      const safe = Array.isArray(parsed) ? parsed : [];
      set({ expenses: safe, hydrated: true, error: undefined });
    } catch {
      set({ hydrated: true, error: 'Failed to load saved expenses.' });
    }
  },

  addExpense: async (expense) => {
    const next = [expense, ...get().expenses];
    set({ expenses: next, error: undefined });
    try {
      await persist(next);
    } catch {
      set({ error: 'Failed to save expense changes.' });
    }
  },

  updateExpense: async (expense) => {
    const next = get().expenses.map((e) => (e.id === expense.id ? expense : e));
    set({ expenses: next, error: undefined });
    try {
      await persist(next);
    } catch {
      set({ error: 'Failed to save expense changes.' });
    }
  },

  deleteExpense: async (id) => {
    const next = get().expenses.filter((e) => e.id !== id);
    set({ expenses: next, error: undefined });
    try {
      await persist(next);
    } catch {
      set({ error: 'Failed to save expense changes.' });
    }
  },
}));
