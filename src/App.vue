<template>
  <div class="app-container">
    <!-- 背景のパーティクル -->
    <div class="particles">
      <div v-for="n in 20" :key="n" class="particle" />
    </div>

    <a href="https://github.com/yamisskey-dev/yamisskey-revision" target="_blank" rel="noopener noreferrer" class="github-corner" aria-label="View source on GitHub">
      <svg width="80" height="80" viewBox="0 0 250 250" style="fill: #86b300; color: #242424; position: fixed; z-index: 10; top: 0; border: 0; right: 0;" aria-hidden="true">
        <path d="M0,0 L115,115 L130,115 L142,142 L250,250 L250,0 Z" />
        <path d="M128.3,109.0 C113.8,99.7 119.0,89.6 119.0,89.6 C122.0,82.7 120.5,78.6 120.5,78.6 C119.2,72.0 123.4,76.3 123.4,76.3 C127.3,80.9 125.5,87.3 125.5,87.3 C122.9,97.6 130.6,101.9 134.4,103.2" fill="currentColor" style="transform-origin: 130px 106px;" class="octo-arm" />
        <path d="M115.0,115.0 C114.9,115.1 118.7,116.5 119.8,115.4 L133.7,101.6 C136.9,99.2 139.9,98.4 142.2,98.6 C133.8,88.0 127.5,74.4 143.8,58.0 C148.5,53.4 154.0,51.2 159.7,51.0 C160.3,49.4 163.2,43.6 171.4,40.1 C171.4,40.1 176.1,42.5 178.8,56.2 C183.1,58.6 187.2,61.8 190.9,65.4 C194.5,69.0 197.7,73.2 200.1,77.6 C213.8,80.2 216.3,84.9 216.3,84.9 C212.7,93.1 206.9,96.0 205.4,96.6 C205.1,102.4 203.0,107.8 198.3,112.5 C181.9,128.9 168.3,122.5 157.7,114.1 C157.9,116.9 156.7,120.9 152.7,124.9 L141.0,136.5 C139.8,137.7 141.6,141.9 141.8,141.8 Z" fill="currentColor" class="octo-body" />
      </svg>
    </a>

    <div class="main-card">
      <!-- アイコン（カード上部に配置） -->
      <div class="icon-container">
        <img
          src="https://raw.githubusercontent.com/misskey-dev/assets/main/icon.png"
          alt="Misskey Logo"
          class="icon-img"
        />
      </div>

      <!-- ヘッダー -->
      <div class="header">
        <h1 class="title">Note Cleaner</h1>
        <p class="subtitle">Misskeyの黒歴史をきれいさっぱり</p>
        <div class="badge">
          <span class="badge-dot" />
          <span>安全に全ノートを削除</span>
        </div>
      </div>

      <!-- フォーム -->
      <form @submit.prevent="handleSubmit" class="form">
        <div class="input-field">
          <label for="host">
            <span class="label-icon">🌐</span>
            サーバーホスト
          </label>
          <input
            id="host"
            v-model="host"
            type="text"
            placeholder="例: misskey.io"
            required
          />
        </div>

        <div class="input-field">
          <label for="token">
            <span class="label-icon">🔑</span>
            APIトークン
          </label>
          <input
            id="token"
            v-model="token"
            type="password"
            placeholder="設定 > API から取得"
            required
          />
        </div>

        <button type="submit" :disabled="isLoading" class="submit-btn">
          <Loader2 v-if="isLoading" class="btn-icon spinning" />
          <Trash2 v-else class="btn-icon" />
          <span>{{ isLoading ? 'クリーニング中...' : 'クリーンアップ開始' }}</span>
        </button>
      </form>

      <!-- エラーメッセージ -->
      <div v-if="error" class="message error">
        <AlertCircle class="message-icon" />
        <p>{{ error }}</p>
      </div>

      <!-- ステータスメッセージ -->
      <div v-if="status" class="message success">
        <CheckCircle2 class="message-icon" />
        <p>{{ status }}</p>
      </div>

      <!-- 進捗バー -->
      <div v-if="isLoading && progress.total > 0" class="progress-area">
        <div class="progress-bar">
          <div
            class="progress-fill"
            :style="{ width: `${(progress.deleted / progress.total) * 100}%` }"
          />
        </div>
        <p class="progress-text">
          {{ progress.deleted }} / {{ progress.total }} ノート削除済み
        </p>
      </div>

    </div>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import { Trash2, AlertCircle, CheckCircle2, Loader2 } from 'lucide-vue-next';
import { MisskeyAPI } from './api';
import type { User, Note } from './types';

const host = ref('');
const token = ref('');
const status = ref('');
const error = ref<string>('');
const isLoading = ref(false);
const progress = ref({ unpinned: 0, deleted: 0, total: 0 });

const handleSubmit = async () => {
  error.value = '';
  status.value = '';
  isLoading.value = true;
  progress.value = { unpinned: 0, deleted: 0, total: 0 };

  try {
    const api = new MisskeyAPI(host.value, token.value);

    const user = await api.getUser();
    status.value = `${user.name ?? user.username} @${user.username} (${user.notesCount} ノート)`;

    if (user.pinnedNotes?.length) {
      for (const note of user.pinnedNotes) {
        if (note.id) {
          await api.unpinNote(note.id);
          progress.value.unpinned++;
        }
      }
    }

    let offset = 0;
    let deleted = 0;
    while (true) {
      const notes = await api.getNotes(user.id!, offset);
      if (!notes.length) break;

      for (const note of notes) {
        if (note.id) {
          try {
            await api.deleteNote(note.id).then(() => deleted++);
            progress.value.deleted = deleted;
            progress.value.total = user.notesCount;
          } catch (error) {
            console.error(`削除失敗: ${note.id}`);
          }
        }
      }

      offset += notes.length;
    }

    status.value = `完了! ${deleted}件のノートを削除しました`;
  } catch (err) {
    error.value = err instanceof Error ? err.message : String(err);
  } finally {
    isLoading.value = false;
  }
};
</script>

<style scoped>
.app-container {
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 1rem;
  background: linear-gradient(135deg, #0f0f0f 0%, #1a1a1a 50%, #0d1117 100%);
  position: relative;
  overflow: hidden;
}

:global(body) {
  margin: 0;
  padding: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

/* パーティクル背景 */
.particles {
  position: absolute;
  inset: 0;
  overflow: hidden;
  pointer-events: none;
}

.particle {
  position: absolute;
  width: 4px;
  height: 4px;
  background: rgba(134, 179, 0, 0.3);
  border-radius: 50%;
  animation: float 15s infinite ease-in-out;
}

.particle:nth-child(1) { left: 10%; top: 20%; animation-delay: 0s; }
.particle:nth-child(2) { left: 20%; top: 80%; animation-delay: 1s; }
.particle:nth-child(3) { left: 30%; top: 40%; animation-delay: 2s; }
.particle:nth-child(4) { left: 40%; top: 60%; animation-delay: 3s; }
.particle:nth-child(5) { left: 50%; top: 30%; animation-delay: 4s; }
.particle:nth-child(6) { left: 60%; top: 70%; animation-delay: 5s; }
.particle:nth-child(7) { left: 70%; top: 50%; animation-delay: 6s; }
.particle:nth-child(8) { left: 80%; top: 20%; animation-delay: 7s; }
.particle:nth-child(9) { left: 90%; top: 90%; animation-delay: 8s; }
.particle:nth-child(10) { left: 15%; top: 50%; animation-delay: 9s; }
.particle:nth-child(11) { left: 25%; top: 10%; animation-delay: 10s; }
.particle:nth-child(12) { left: 35%; top: 90%; animation-delay: 11s; }
.particle:nth-child(13) { left: 45%; top: 15%; animation-delay: 12s; }
.particle:nth-child(14) { left: 55%; top: 85%; animation-delay: 13s; }
.particle:nth-child(15) { left: 65%; top: 25%; animation-delay: 14s; }
.particle:nth-child(16) { left: 75%; top: 75%; animation-delay: 0.5s; }
.particle:nth-child(17) { left: 85%; top: 35%; animation-delay: 1.5s; }
.particle:nth-child(18) { left: 95%; top: 55%; animation-delay: 2.5s; }
.particle:nth-child(19) { left: 5%; top: 65%; animation-delay: 3.5s; }
.particle:nth-child(20) { left: 50%; top: 50%; animation-delay: 4.5s; }

@keyframes float {
  0%, 100% {
    transform: translateY(0) scale(1);
    opacity: 0.3;
  }
  50% {
    transform: translateY(-100px) scale(1.5);
    opacity: 0.8;
  }
}

/* メインカード */
.main-card {
  background: rgba(36, 36, 36, 0.95);
  backdrop-filter: blur(20px);
  border-radius: 24px;
  padding: 2.5rem;
  padding-top: 3.5rem;
  margin-top: 50px;
  max-width: 420px;
  width: 100%;
  position: relative;
  border: 1px solid rgba(134, 179, 0, 0.2);
  box-shadow:
    0 4px 24px rgba(0, 0, 0, 0.4),
    0 0 0 1px rgba(255, 255, 255, 0.05) inset,
    0 0 80px rgba(134, 179, 0, 0.1);
}

/* アイコンエリア（カード上部に配置） */
.icon-container {
  position: absolute;
  top: -40px;
  left: 50%;
  transform: translateX(-50%);
}

.icon-img {
  width: 80px;
  height: 80px;
  object-fit: contain;
}

/* ヘッダー */
.header {
  text-align: center;
  margin-bottom: 2rem;
}

.title {
  font-size: 2rem;
  font-weight: 800;
  color: #fff;
  margin: 0 0 0.5rem 0;
  letter-spacing: -0.02em;
}

.subtitle {
  color: rgba(255, 255, 255, 0.6);
  font-size: 0.95rem;
  margin: 0 0 1rem 0;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(134, 179, 0, 0.1);
  border: 1px solid rgba(134, 179, 0, 0.25);
  border-radius: 100px;
  padding: 0.5rem 1rem;
  font-size: 0.85rem;
  color: #86b300;
}

.badge-dot {
  width: 8px;
  height: 8px;
  background: #86b300;
  border-radius: 50%;
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; transform: scale(1); }
  50% { opacity: 0.5; transform: scale(0.8); }
}

/* フォーム */
.form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.input-field {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.input-field label {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.9rem;
  color: rgba(255, 255, 255, 0.8);
  font-weight: 500;
}

.label-icon {
  font-size: 1rem;
}

.input-field input {
  width: 100%;
  padding: 0.875rem 1rem;
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  color: #fff;
  font-size: 1rem;
  transition: all 0.2s ease;
  box-sizing: border-box;
}

.input-field input::placeholder {
  color: rgba(255, 255, 255, 0.3);
}

.input-field input:focus {
  outline: none;
  border-color: #86b300;
  background: rgba(134, 179, 0, 0.08);
  box-shadow: 0 0 0 3px rgba(134, 179, 0, 0.15);
}

/* ボタン */
.submit-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.75rem;
  width: 100%;
  padding: 1rem;
  margin-top: 0.5rem;
  background: linear-gradient(135deg, #86b300 0%, #638506 100%);
  border: none;
  border-radius: 12px;
  color: #0f0f0f;
  font-size: 1rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 4px 15px rgba(134, 179, 0, 0.3);
}

.submit-btn:hover:not(:disabled) {
  transform: translateY(-2px);
  box-shadow: 0 6px 25px rgba(134, 179, 0, 0.4);
}

.submit-btn:active:not(:disabled) {
  transform: translateY(0);
}

.submit-btn:disabled {
  opacity: 0.7;
  cursor: not-allowed;
}

.btn-icon {
  width: 20px;
  height: 20px;
}

.spinning {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

/* メッセージ */
.message {
  display: flex;
  align-items: flex-start;
  gap: 0.75rem;
  padding: 1rem;
  border-radius: 12px;
  margin-top: 1.25rem;
  font-size: 0.9rem;
}

.message p {
  margin: 0;
  white-space: pre-line;
}

.message-icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
  margin-top: 2px;
}

.message.error {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.2);
  color: #fca5a5;
}

.message.error .message-icon {
  color: #ef4444;
}

.message.success {
  background: rgba(134, 179, 0, 0.1);
  border: 1px solid rgba(134, 179, 0, 0.2);
  color: #c6f6a0;
}

.message.success .message-icon {
  color: #86b300;
}

/* 進捗エリア */
.progress-area {
  margin-top: 1.5rem;
}

.progress-bar {
  height: 8px;
  background: rgba(134, 179, 0, 0.15);
  border-radius: 100px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #86b300, #aadc06);
  border-radius: 100px;
  transition: width 0.3s ease;
}

.progress-text {
  text-align: center;
  color: rgba(255, 255, 255, 0.6);
  font-size: 0.85rem;
  margin: 0.75rem 0 0 0;
}

/* レスポンシブ */
@media (max-width: 480px) {
  .main-card {
    padding: 1.5rem;
    border-radius: 20px;
  }

  .title {
    font-size: 1.6rem;
  }

  .icon-container {
    top: -32px;
  }

  .icon-img {
    width: 64px;
    height: 64px;
  }
}
</style>

<style>
.github-corner:hover .octo-arm {
  animation: octocat-wave 560ms ease-in-out;
}

@keyframes octocat-wave {
  0%, 100% { transform: rotate(0); }
  20%, 60% { transform: rotate(-25deg); }
  40%, 80% { transform: rotate(10deg); }
}

@media (max-width: 500px) {
  .github-corner:hover .octo-arm {
    animation: none;
  }
  .github-corner .octo-arm {
    animation: octocat-wave 560ms ease-in-out;
  }
}
</style>
