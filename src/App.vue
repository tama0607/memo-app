<template>
  <div id="app">
    <h1>Memo App</h1>

    <!-- トークン入力フィールド -->
    <div>
      <input
        type="text"
        id="access_token"
        v-model="accessToken"
        placeholder="Enter Access Token"
        :disabled="isAccessTokenDisabled"
      />
      <button
        id="login"
        :disabled="!isValidToken || isAccessTokenDisabled"
        @click="handleLogin"
      >
        LOGIN
      </button>
    </div>

    <!-- カテゴリ一覧 -->
    <div v-if="categories.length > 0" class="category-container">
      <div
        v-for="category in categories"
        :key="category.id"
        :id="'category-' + category.id"
        class="category"
      >
        <!-- カテゴリタイトル -->
        <div
          :id="'category-' + category.id + '-title'"
          class="category-title"
          @click="toggleCategory(category.id)"
        >
          <span class="category-title-icon">📁</span>
          {{ category.name }}
        </div>

        <!-- メモ一覧（展開状態のときのみ表示） -->
        <div v-if="expandedCategory === category.id" class="memo-list">
          <div
            v-for="memo in memos[category.id] || []"
            :key="memo.id"
            :id="'memo-' + memo.id"
            class="memo-item"
            @click="selectMemo(memo.id)"
          >
            <span class="memo-item-icon">📄</span>
            {{ memo.title }}
          </div>
        </div>
      </div>

      <!-- メモ追加ボタン（カテゴリが展開された場合に表示） -->
      <button
        id="new-memo"
        :disabled="!isCategoryExpanded"
        @click="addMemo"
        class="new-memo-button"
      >
        NEW
      </button>

      <!-- メモ削除ボタン（選択状態のときのみ有効） -->
      <button
        id="delete-memo"
        :disabled="!isMemoSelected"
        @click="deleteMemo"
        class="delete-memo-button"
      >
        DELETE
      </button>
    </div>

    <!-- メモ編集 -->
    <div class="memo-editor">
      <input
        type="text"
        id="memo-title"
        v-model="selectedMemo.title"
        placeholder="Memo Title"
        :disabled="!isMemoSelected"
      />
      <textarea
        id="memo-content"
        v-model="selectedMemo.content"
        placeholder="Memo Content"
        :disabled="!isMemoSelected"
      ></textarea>
      <button
        id="save-memo"
        :disabled="!isMemoSelected"
        @click="saveMemo"
      >
        SAVE
      </button>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from "vue";

export default {
  setup() {
    const API_BASE_URL = "https://challenge-server.tracks.run/memoapp";

    const accessToken = ref(""); // トークン
    const isAccessTokenDisabled = ref(false); // トークン入力とボタンの無効状態
    const categories = ref([]); // カテゴリ一覧
    const memos = ref({}); // メモ一覧（カテゴリIDごとに格納）
    const expandedCategory = ref(null); // 現在展開中のカテゴリID
    const selectedMemo = ref({ title: "", content: "", id: null }); // 現在選択されたメモ
    const isMemoSelected = computed(() => selectedMemo.value.id !== null); // メモ選択状態
    const isCategoryExpanded = computed(() => expandedCategory.value !== null); // カテゴリ展開状態

    const uuidRegex =
      /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;

    const isValidToken = computed(() => uuidRegex.test(accessToken.value));

    // UUIDを生成する関数
    const generateUUID = () => {
      return "xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx".replace(/[xy]/g, (c) => {
        const r = (Math.random() * 16) | 0;
        const v = c === "x" ? r : (r & 0x3) | 0x8;
        return v.toString(16);
      });
    };

    // 初期化処理：ページロード後にアクセストークンを動的に生成
    onMounted(() => {
      setTimeout(() => {
        accessToken.value = generateUUID();
      }, 1000);
    });

    // カテゴリ一覧を取得
    const fetchCategories = async () => {
      try {
        const response = await fetch(`${API_BASE_URL}/category`, {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            "X-ACCESS-TOKEN": accessToken.value,
          },
        });

        if (!response.ok) throw new Error("Failed to fetch categories");

        const data = await response.json();
        categories.value = data;
      } catch (error) {
        console.error(error);
      }
    };

    // メモ一覧を取得
    const fetchMemos = async (categoryId) => {
      try {
        const response = await fetch(
          `${API_BASE_URL}/memo?category_id=${categoryId}`,
          {
            method: "GET",
            headers: {
              "Content-Type": "application/json",
              "X-ACCESS-TOKEN": accessToken.value,
            },
          }
        );

        if (!response.ok) throw new Error("Failed to fetch memos");

        const data = await response.json();
        memos.value[categoryId] = data;
      } catch (error) {
        console.error(error);
      }
    };

    // メモの選択処理
    const selectMemo = async (memoId) => {
      try {
        const response = await fetch(`${API_BASE_URL}/memo/${memoId}`, {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
            "X-ACCESS-TOKEN": accessToken.value,
          },
        });

        if (!response.ok) throw new Error("Failed to fetch memo");

        const data = await response.json();
        selectedMemo.value = {
          id: data.id,
          title: data.title,
          content: data.content,
        };
      } catch (error) {
        console.error(error);
      }
    };

    // メモの保存処理
    const saveMemo = async () => {
      if (!isMemoSelected.value) return;

      try {
        const response = await fetch(`${API_BASE_URL}/memo/${selectedMemo.value.id}`, {
          method: "PUT",
          headers: {
            "Content-Type": "application/json",
            "X-ACCESS-TOKEN": accessToken.value,
          },
          body: JSON.stringify({
            category_id: expandedCategory.value,
            title: selectedMemo.value.title,
            content: selectedMemo.value.content,
          }),
        });

        if (!response.ok) throw new Error("Failed to update memo");

        const data = await response.json();
        console.log("Memo updated:", data);

        // 更新後、メモリストを更新
        const categoryMemos = memos.value[expandedCategory.value] || [];
        const index = categoryMemos.findIndex((m) => m.id === selectedMemo.value.id);
        if (index !== -1) {
          categoryMemos[index] = { ...categoryMemos[index], title: data.title };
        }
      } catch (error) {
        console.error(error);
      }
    };

    // メモ追加処理
    const addMemo = async () => {
      if (!isCategoryExpanded.value) return;

      try {
        const response = await fetch(`${API_BASE_URL}/memo`, {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            "X-ACCESS-TOKEN": accessToken.value,
          },
          body: JSON.stringify({
            category_id: expandedCategory.value,
            title: "New Memo",
            content: "New content",
          }),
        });

        if (!response.ok) throw new Error("Failed to add memo");

        const data = await response.json();
        // 追加したメモをリストに反映
        if (!memos.value[expandedCategory.value]) {
          memos.value[expandedCategory.value] = [];
        }
        memos.value[expandedCategory.value].push(data);

        // 追加したメモを選択状態にする
        selectedMemo.value = {
          id: data.id,
          title: data.title,
          content: data.content,
        };
      } catch (error) {
        console.error(error);
      }
    };

    // メモ削除処理
    const deleteMemo = async () => {
      if (!isMemoSelected.value) return;

      try {
        const response = await fetch(`${API_BASE_URL}/memo/${selectedMemo.value.id}`, {
          method: "DELETE",
          headers: {
            "Content-Type": "application/json",
            "X-ACCESS-TOKEN": accessToken.value,
          },
        });

        if (!response.ok) throw new Error("Failed to delete memo");

        // 削除後、メモリストから該当のメモを削除
        const categoryMemos = memos.value[expandedCategory.value] || [];
        const index = categoryMemos.findIndex((m) => m.id === selectedMemo.value.id);
        if (index !== -1) {
          categoryMemos.splice(index, 1);
        }

        // メモ選択解除
        selectedMemo.value = { title: "", content: "", id: null };
      } catch (error) {
        console.error(error);
      }
    };

    // カテゴリの展開・縮小を切り替え
    const toggleCategory = async (categoryId) => {
      if (expandedCategory.value === categoryId) {
        expandedCategory.value = null;
      } else {
        expandedCategory.value = categoryId;
        await fetchMemos(categoryId);
      }
    };

    // ログイン処理
    const handleLogin = async () => {
      isAccessTokenDisabled.value = true;
      await fetchCategories();
    };

    return {
      accessToken,
      isAccessTokenDisabled,
      isValidToken,
      categories,
      memos,
      expandedCategory,
      selectedMemo,
      isMemoSelected,
      isCategoryExpanded,
      addMemo,
      selectMemo,
      saveMemo,
      deleteMemo,
      toggleCategory,
      handleLogin,
    };
  },
};
</script>

<style scoped>
/* 全体の基本スタイル */
#app {
  font-family: Arial, sans-serif;
  margin: 20px;
  text-align: center;
}

/* トークン入力フィールド */
input[type="text"] {
  margin-right: 10px;
  padding: 8px;
  font-size: 14px;
  width: 300px;
}

button {
  padding: 8px 16px;
  font-size: 14px;
  cursor: pointer;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  transition: background-color 0.3s ease;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

button:hover:not(:disabled) {
  background-color: #0056b3;
}

/* カテゴリ全体のスタイル */
.category-container {
  position: relative;
}

.category {
  border: 1px solid #ddd;
  border-radius: 5px;
  margin: 10px 0;
  padding: 10px;
  background-color: #fff;
  cursor: pointer;
  display: flex;
  flex-direction: column;
  transition: background-color 0.3s ease;
}

.category:hover {
  background-color: #f9f9f9;
}

/* メモ追加ボタン */
#new-memo,
#delete-memo {
  padding: 8px 16px;
  font-size: 14px;
  cursor: pointer;
  background-color: #28a745;
  color: white;
  border: none;
  border-radius: 5px;
  transition: background-color 0.3s ease;
}

#new-memo:disabled,
#delete-memo:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

#new-memo:hover:not(:disabled),
#delete-memo:hover:not(:disabled) {
  background-color: #218838;
}

/* メモ編集フォーム */
.memo-editor {
  margin-top: 20px;
  text-align: left;
  max-width: 400px;
  margin: 20px auto;
}

.memo-editor input,
.memo-editor textarea {
  border: 1px solid #ccc;
  border-radius: 5px;
  margin: 10px 0;
  padding: 10px;
  font-size: 14px;
  width: 100%;
  box-sizing: border-box;
  display: block;
}

.memo-editor textarea {
  height: 100px;
  resize: none;
}

.memo-editor button {
  width: 100%;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  padding: 10px 15px;
  font-size: 16px;
}

.memo-editor button:hover:not(:disabled) {
  background-color: #0056b3;
}

.memo-editor button:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}
</style>
