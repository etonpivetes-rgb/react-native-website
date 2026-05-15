import React, { useEffect, useMemo, useState } from 'react';
import {
  SafeAreaView,
  View,
  Text,
  TextInput,
  TouchableOpacity,
  FlatList,
  Alert,
  StyleSheet,
  Modal,
  Pressable,
} from 'react-native';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { StatusBar } from 'expo-status-bar';

const STORAGE_KEY = '@finance_app_transactions';
const categories = ['Alimentação', 'Transporte', 'Casa', 'Lazer', 'Saúde', 'Outros'];

export default function App() {
  const [transactions, setTransactions] = useState([]);
  const [modalVisible, setModalVisible] = useState(false);
  const [editingId, setEditingId] = useState(null);
  const [form, setForm] = useState({
    description: '',
    amount: '',
    type: 'débito',
    category: 'Outros',
  });

  useEffect(() => {
    loadTransactions();
  }, []);

  useEffect(() => {
    AsyncStorage.setItem(STORAGE_KEY, JSON.stringify(transactions));
  }, [transactions]);

  const loadTransactions = async () => {
    const data = await AsyncStorage.getItem(STORAGE_KEY);
    if (data) setTransactions(JSON.parse(data));
  };

  const resetForm = () => {
    setForm({
      description: '',
      amount: '',
      type: 'débito',
      category: 'Outros',
    });
    setEditingId(null);
  };

  const saveTransaction = () => {
    if (!form.description.trim() || !form.amount) {
      Alert.alert('Atenção', 'Preencha descrição e valor.');
      return;
    }

    const item = {
      id: editingId ?? Date.now().toString(),
      description: form.description.trim(),
      amount: Number(form.amount),
      type: form.type,
      category: form.category,
      date: new Date().toISOString(),
    };

    if (editingId) {
      setTransactions(prev =>
        prev.map(t => (t.id === editingId ? item : t))
      );
    } else {
      setTransactions(prev => [item, ...prev]);
    }

    setModalVisible(false);
    resetForm();
  };

  const deleteTransaction = (id) => {
    Alert.alert('Excluir', 'Deseja excluir este gasto?', [
      { text: 'Cancelar', style: 'cancel' },
      {
        text: 'Excluir',
        style: 'destructive',
        onPress: () =>
          setTransactions(prev => prev.filter(t => t.id !== id)),
      },
    ]);
  };

  const editTransaction = (item) => {
    setForm({
      description: item.description,
      amount: String(item.amount),
      type: item.type,
      category: item.category,
    });
    setEditingId(item.id);
    setModalVisible(true);
  };

  const totals = useMemo(() => {
    const credit = transactions
      .filter(t => t.type === 'crédito')
      .reduce((sum, t) => sum + t.amount, 0);

    const debit = transactions
      .filter(t => t.type === 'débito')
      .reduce((sum, t) => sum + t.amount, 0);

    return {
      credit,
      debit,
      total: credit + debit,
    };
  }, [transactions]);

  const renderItem = ({ item }) => (
    <View style={styles.card}>
      <View style={{ flex: 1 }}>
        <Text style={styles.title}>{item.description}</Text>
        <Text style={styles.subtitle}>
          {item.category} • {item.type}
        </Text>
      </View>

      <View style={{ alignItems: 'flex-end' }}>
        <Text style={styles.amount}>R$ {item.amount.toFixed(2)}</Text>

        <View style={styles.actions}>
          <TouchableOpacity
            onPress={() => editTransaction(item)}
            style={styles.actionBtn}
          >
            <Text style={styles.actionText}>Editar</Text>
          </TouchableOpacity>

          <TouchableOpacity
            onPress={() => deleteTransaction(item.id)}
            style={styles.actionBtn}
          >
            <Text style={styles.actionText}>Excluir</Text>
          </TouchableOpacity>
        </View>
      </View>
    </View>
  );

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar style="light" />

      <Text style={styles.header}>Controle Financeiro</Text>

      <View style={styles.summary}>
        <Text style={styles.summaryText}>
          Total geral: R$ {totals.total.toFixed(2)}
        </Text>
        <Text style={styles.summaryText}>
          Crédito: R$ {totals.credit.toFixed(2)}
        </Text>
        <Text style={styles.summaryText}>
          Débito: R$ {totals.debit.toFixed(2)}
        </Text>
      </View>

      <TouchableOpacity
        style={styles.addBtn}
        onPress={() => {
          resetForm();
          setModalVisible(true);
        }}
      >
        <Text style={styles.addBtnText}>+ Novo gasto</Text>
      </TouchableOpacity>

      <FlatList
        data={transactions}
        keyExtractor={item => item.id}
        renderItem={renderItem}
        contentContainerStyle={{ paddingBottom: 24 }}
        ListEmptyComponent={
          <Text style={styles.empty}>Nenhum gasto cadastrado.</Text>
        }
      />

      <Modal visible={modalVisible} animationType="slide" transparent>
        <View style={styles.modalOverlay}>
          <View style={styles.modalCard}>
            <Text style={styles.modalTitle}>
              {editingId ? 'Editar gasto' : 'Novo gasto'}
            </Text>

            <TextInput
              placeholder="Descrição"
              value={form.description}
              onChangeText={description => setForm({ ...form, description })}
              style={styles.input}
            />

            <TextInput
              placeholder="Valor"
              keyboardType="numeric"
              value={form.amount}
              onChangeText={amount => setForm({ ...form, amount })}
              style={styles.input}
            />

            <View style={styles.row}>
              {['débito', 'crédito'].map(type => (
                <Pressable
                  key={type}
                  onPress={() => setForm({ ...form, type })}
                  style={[
                    styles.pill,
                    form.type === type && styles.pillActive,
                  ]}
                >
                  <Text style={styles.pillText}>{type}</Text>
                </Pressable>
              ))}
            </View>

            <View style={styles.rowWrap}>
              {categories.map(cat => (
                <Pressable
                  key={cat}
                  onPress={() => setForm({ ...form, category: cat })}
                  style={[
                    styles.pill,
                    form.category === cat && styles.pillActive,
                  ]}
                >
                  <Text style={styles.pillText}>{cat}</Text>
                </Pressable>
              ))}
            </View>

            <View style={styles.row}>
              <TouchableOpacity
                onPress={() => {
                  setModalVisible(false);
                  resetForm();
                }}
                style={[styles.modalBtn, styles.cancelBtn]}
              >
                <Text style={styles.btnText}>Cancelar</Text>
              </TouchableOpacity>

              <TouchableOpacity
                onPress={saveTransaction}
                style={[styles.modalBtn, styles.saveBtn]}
              >
                <Text style={styles.btnText}>Salvar</Text>
              </TouchableOpacity>
            </View>
          </View>
        </View>
      </Modal>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#0f172a',
    padding: 16,
  },
  header: {
    color: '#fff',
    fontSize: 28,
    fontWeight: '700',
    marginBottom: 12,
  },
  summary: {
    backgroundColor: '#1e293b',
    padding: 16,
    borderRadius: 16,
    marginBottom: 16,
  },
  summaryText: {
    color: '#e2e8f0',
    fontSize: 16,
    marginBottom: 4,
  },
  addBtn: {
    backgroundColor: '#22c55e',
    padding: 14,
    borderRadius: 14,
    alignItems: 'center',
    marginBottom: 16,
  },
  addBtnText: {
    color: '#fff',
    fontWeight: '700',
  },
  card: {
    backgroundColor: '#1e293b',
    padding: 14,
    borderRadius: 16,
    flexDirection: 'row',
    marginBottom: 12,
    gap: 10,
  },
  title: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '700',
  },
  subtitle: {
    color: '#94a3b8',
    marginTop: 4,
  },
  amount: {
    color: '#f8fafc',
    fontSize: 16,
    fontWeight: '700',
  },
  actions: {
    flexDirection: 'row',
    gap: 8,
    marginTop: 8,
  },
  actionBtn: {
    backgroundColor: '#334155',
    paddingHorizontal: 10,
    paddingVertical: 6,
    borderRadius: 10,
  },
  actionText: {
    color: '#fff',
    fontSize: 12,
  },
  empty: {
    color: '#94a3b8',
    textAlign: 'center',
    marginTop: 30,
  },
  modalOverlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.6)',
    justifyContent: 'center',
    padding: 16,
  },
  modalCard: {
    backgroundColor: '#fff',
    borderRadius: 20,
    padding: 16,
  },
  modalTitle: {
    fontSize: 20,
    fontWeight: '700',
    marginBottom: 12,
  },
  input: {
    borderWidth: 1,
    borderColor: '#cbd5e1',
    borderRadius: 12,
    padding: 12,
    marginBottom: 12,
  },
  row: {
    flexDirection: 'row',
    gap: 10,
    marginBottom: 12,
    flexWrap: 'wrap',
  },
  rowWrap: {
    flexDirection: 'row',
    gap: 10,
    marginBottom: 12,
    flexWrap: 'wrap',
  },
  pill: {
    paddingHorizontal: 12,
    paddingVertical: 8,
    borderRadius: 999,
    backgroundColor: '#e2e8f0',
  },
  pillActive: {
    backgroundColor: '#0f172a',
  },
  pillText: {
    color: '#111827',
  },
  modalBtn: {
    flex: 1,
    padding: 14,
    borderRadius: 12,
    alignItems: 'center',
  },
  cancelBtn: {
    backgroundColor: '#64748b',
  },
  saveBtn: {
    backgroundColor: '#16a34a',
  },
  btnText: {
    color: '#fff',
    fontWeight: '700',
  },
});
