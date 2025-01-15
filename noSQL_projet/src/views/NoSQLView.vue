<script lang="ts">
import { ref } from "vue";
import PouchDB from "pouchdb";

declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
  };
}

interface PostRow {
  _id: string;
  doc: Post;
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as PostRow[],
      document: {
        post_name: "",
        post_content: "",
        attributes: {
          creation_date: new Date().toISOString(),
        },
      },
      editingPost: null as Post | null,
      storage: null as PouchDB.Database | null,
    };
  },

  mounted() {
    this.initDatabase();
    this.fetchData();
  },

  methods: {
    fetchData() {
      const storage = ref(this.storage);
      const self = this;
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true,
          })
          .then(function (result: any) {
            console.log("fetchData success", result);
            self.postsData = result.rows.map((row: any) => {
              return {
                _id: row.id,
                doc: row.doc,
              };
            });
          })
          .catch(function (error: any) {
            console.log("fetchData error", error);
          });
      }
    },

    async createDocument() {
      const db = ref(this.storage).value;
      const doc = {
        post_name: this.document.post_name,
        post_content: this.document.post_content,
        attributes: {
          creation_date: new Date().toISOString(),
        },
      };
      try {
        if (db != null) {
          const response = await db.post(doc);
          if (response.ok) {
            this.document = {
              post_name: "",
              post_content: "",
              attributes: {
                creation_date: new Date().toISOString(),
              },
            };
            this.fetchData();
            return response.ok;
          }
        }
      } catch (err) {
        console.log(err);
      }
    },

    updateDocument(id: string) {
      const post = this.postsData.find((post) => post._id === id);
      if (post) {
        this.editingPost = { ...post.doc };
      }
    },

    async saveUpdate(post: Post) {
      const db = ref(this.storage).value;
      if (!db || !post._id) return;

      try {
        const existingDoc = await db.get(post._id);
        const updatedDoc = {
          _id: post._id,
          _rev: existingDoc._rev,
          post_name: post.post_name,
          post_content: post.post_content,
          attributes: {
            creation_date: post.attributes.creation_date
          }
        };

        const response = await db.put(updatedDoc);
        if (response.ok) {
          this.editingPost = null;
          this.fetchData();
        }
      } catch (err) {
        console.log('Update error:', err);
      }
    },

    async deleteDocument(id: string) {
      const db = ref(this.storage).value;
      if (!db) return;

      try {
        const doc = await db.get(id);
        const response = await db.remove(doc);
        if (response.ok) {
          this.fetchData();
        }
      } catch (err) {
        console.log('Delete error:', err);
      }
    },

    cancelEdit() {
      this.editingPost = null;
    },

    initDatabase() {
      const db = new PouchDB("LocalDB");
      if (db) {
        //http://localhost:5984/_utils/#login
          const remoteDB = new PouchDB("http://admin:couchdb@localhost:5984/post");
          if(remoteDB){
            db.sync(remoteDB, {
              live: true,
              retry: true,
            });
          }else{
          console.warn("Remote database not found");
        }
        console.log("Connected to collection 'post'");
      } else {
        console.warn("Something went wrong");
      }
      this.storage = db;
    },
  },
};
</script>

<template>
  <div>
    <form @submit.prevent="createDocument">
      <div>
        <label for="title">Title:</label>
        <input id="title" v-model="document.post_name" required />
      </div>
      <div>
        <label for="content">Content:</label>
        <textarea
          id="content"
          v-model="document.post_content"
          required
        ></textarea>
      </div>
      <button type="submit">Create Document</button>
    </form>
  </div>

  <div>
    <h2>Posts</h2>
    <div v-if="postsData.length === 0">No documents found.</div>
    <div v-else>
      <div v-for="post in postsData" :key="post._id" class="document-item">
        <!-- Display mode -->
        <div v-if="editingPost?._id !== post._id">
          <h3>{{ post.doc.post_name }}</h3>
          <p>{{ post.doc.post_content }}</p>
          <p>
            Created:
            {{ new Date(post.doc.attributes?.creation_date).toLocaleString() }}
          </p>
          <button
            @click="deleteDocument(post._id)"
            class="delete-btn"
          >
            Delete
          </button>
          <button
            @click="updateDocument(post._id)"
            class="update-btn"
          >
            Edit
          </button>
        </div>

        <!-- Edit mode -->
        <div v-else class="edit-form">
          <form @submit.prevent="saveUpdate(editingPost)">
            <div>
              <label :for="'edit-title-' + post._id">Title:</label>
              <input
                :id="'edit-title-' + post._id"
                v-model="editingPost.post_name"
                required
              />
            </div>
            <div>
              <label :for="'edit-content-' + post._id">Content:</label>
              <textarea
                :id="'edit-content-' + post._id"
                v-model="editingPost.post_content"
                required
              ></textarea>
            </div>
            <button type="submit">Save</button>
            <button type="button" @click="cancelEdit">Cancel</button>
          </form>
        </div>
        <hr />
      </div>
    </div>
  </div>
</template>

<style scoped>
.delete-btn {
  background-color: #ff4444;
  color: white;
  padding: 5px 10px;
  margin-right: 5px;
}

.update-btn {
  background-color: #44aaff;
  color: white;
  padding: 5px 10px;
}

.edit-form {
  margin: 1rem 0;
  padding: 1rem;
  background-color: #f5f5f5;
  border-radius: 4px;
  color:black;
}

form div {
  margin-bottom: 1rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
}

input, textarea {
  width: 100%;
  padding: 0.5rem;
}

button {
  margin-right: 0.5rem;
}

textarea {
  min-height: 100px;
}
</style>