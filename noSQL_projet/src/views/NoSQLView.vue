<script lang="ts">
import { ref } from "vue";
import PouchDB from "pouchdb";
import find from "pouchdb-find";
PouchDB.plugin(find);

declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  _attachments?: {
    [key: string]: {
      content_type: string;
      digest: string;
      length: number;
      stub: boolean;
    };
  };
  attributes: {
    creation_date: string;
  };
}

interface PostRow {
  _id: string;
  doc: Post;
  attachments?: {
    filename: string;
    url: string;
    contentType: string;
  }[];
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as PostRow[],
      searchResults: [] as Post[],
      searchQuery: "",
      startDate: "",
      endDate: "",
      isSearchActive: false,
      document: {
        post_name: "",
        post_content: "",
        attributes: {
          creation_date: new Date().toISOString(),
        },
      },
      selectedFile: null as File | null,
      editingPost: null as Post | null,
      storage: null as PouchDB.Database | null,
    };
  },

  async mounted() {
    await this.initDatabase();
    // const db = ref(this.storage).value;
    // if (db){
    //   this.generePosts(db).then(() => {
    this.fetchData();
    //   });
    // }
  },

  methods: {
    fetchData() {
      const storage = ref(this.storage);
      const self = this;

      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true, // Inclure les documents
            attachments: false, // Ne pas inclure les données des pièces jointes ici
          })
          .then(async function (result: any) {
            console.log("fetchData success", result);
            self.postsData = result.rows.map((row: any) => ({
              _id: row.id,
              doc: row.doc,
            }));
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
            creation_date: post.attributes.creation_date,
          },
        };

        const response = await db.put(updatedDoc);
        if (response.ok) {
          this.editingPost = null;
          this.fetchData();
        }
      } catch (err) {
        console.log("Update error:", err);
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
        console.log("Delete error:", err);
      }
    },

    cancelEdit() {
      this.editingPost = null;
    },
    /**
     * Fonction pour générer des posts 100 aléatoires
     * @param db Base de donnée
     */
    async generePosts(db: PouchDB.Database) {
      const categories = [
        "Science",
        "Art",
        "Design",
        "Architecture",
        "Skate",
        "Musique",
        "Manga",
        "Voyage",
        "Sport",
        "Cuisine",
      ];
      try {
        for (let i = 0; i < 100; i++) {
          const category =
            categories[Math.floor(Math.random() * categories.length)];
          const postNumber = (i + 1).toString().padStart(3, "0");
          const content = [
            category + " post " + postNumber,
            "Lorem ipsum dolor sit amet, consectetur adipiscing elit.",
            "Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.",
            "Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat.",
            "Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.",
          ].join(" ");
          const post = {
            post_name: `${category}-Post-${postNumber}`,
            post_content: content,
            attributes: {
              creation_date: new Date().toISOString(),
            },
          };

          await db.post(post);
          console.log(`Created post ${i + 1}/100: ${post.post_name}`);
        }

        console.log("Successfully generated 100 posts");
        return true;
      } catch (error) {
        console.error("Error generating posts:", error);
        return false;
      }
    },
    async searchByName() {
      if (!this.searchQuery.trim()) {
        this.clearSearch();
        return;
      }

      const db = ref(this.storage).value;
      if (!db) return;

      try {
        const result = await db.find({
          selector: {
            post_name: { $regex: RegExp(this.searchQuery, "i") },
          },
          use_index: "post-name-index",
        });
        console.log("Search result:", result);
        this.searchResults = result.docs;
        this.isSearchActive = true;
      } catch (error) {
        console.error("Search error:", error);
        this.searchResults = [];
      }
    },
    clearSearch() {
      this.searchResults = [];
      this.searchQuery = "";
      this.isSearchActive = false;
    },

    async addAttachment(postId: string, file: File) {
      const db = ref(this.storage).value;
      if (!db) {
        console.error("Database not initialized");
        return;
      }

      try {
        const doc = await db.get(postId); // Get current document
        await db.putAttachment(postId, file.name, doc._rev, file, file.type);
        console.log("Attachment added successfully");
        await this.fetchData(); // Reload all data
      } catch (error) {
        console.error("Error adding attachment:", error);
      }
    },
    handleFileUpload(event: Event) {
      const target = event.target as HTMLInputElement;
      if (target.files && target.files.length > 0) {
        this.selectedFile = target.files[0];
      }
    },

    async addAttachmentToPost(postId: string) {
      if (!this.selectedFile) {
        alert("Veuillez sélectionner un fichier.");
        return;
      }
      await this.addAttachment(postId, this.selectedFile);
      this.selectedFile = null; // Réinitialiser après l'ajout
    },
    async fetchAttachments(postId: string) {
      const db = ref(this.storage).value;
      if (!db) return [];

      try {
        // Get the document with attachments
        const doc = await db.get(postId, { attachments: true });
        if (!doc._attachments) return [];

        // Convert each attachment to a downloadable format
        return await Promise.all(
          Object.entries(doc._attachments).map(
            async ([filename, attachment]) => {
              const blob = await db.getAttachment(postId, filename);
              return {
                filename,
                url: URL.createObjectURL(blob), // Create downloadable URL
                contentType: attachment.content_type,
              };
            }
          )
        );
      } catch (error) {
        console.error("Error fetching attachments:", error);
        return [];
      }
    },
    async initDatabase() {
      const db = new PouchDB("LocalDB");
      if (db) {
        //http://localhost:5984/_utils/#login
        const remoteDB = new PouchDB(
          "http://admin:couchdb@localhost:5984/post"
        );
        if (remoteDB) {
          db.sync(remoteDB, {
            live: true,
            retry: true,
          });

          try {
            // Create indexes
            await db.createIndex({
              index: {
                fields: ["post_name"],
                name: "post-name-index",
                ddoc: "post-name-index",
              },
            });
            console.log("Created post name index");
          } catch (error) {
            console.error("Error creating indexes:", error);
          }
        } else {
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
  <div class="container">
    <!-- Search Section -->
    <div class="search-section">
      <h2 class="searchTitle">Search Posts</h2>

      <!-- Search by name -->
      <div class="search-box">
        <input
          v-model="searchQuery"
          @input="searchByName"
          placeholder="Search by post name..."
          class="search-input"
        />
      </div>

      <button v-if="isSearchActive" @click="clearSearch" class="clear-btn">
        Clear Search
      </button>
    </div>

    <!-- Create Post Form -->
    <div class="create-section">
      <h2>Create New Post</h2>
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
        <button type="submit" class="create-btn">Create Document</button>
      </form>
    </div>

    <!-- Posts Display -->
    <div class="posts-section">
      <h2>{{ isSearchActive ? "Search Results" : "All Posts" }}</h2>
      <div
        v-if="isSearchActive && searchResults.length === 0"
        class="no-results"
      >
        No posts found matching your search criteria.
      </div>
      <div
        v-else-if="!isSearchActive && postsData.length === 0"
        class="no-results"
      >
        No documents found.
      </div>
      <div v-else>
        <div
          v-for="post in isSearchActive ? searchResults : postsData"
          :key="post._id"
          class="document-item"
        >
          <!-- Display mode -->
          <div v-if="editingPost?._id !== post._id">
            <h3>{{ isSearchActive ? post.post_name : post.doc.post_name }}</h3>
            <p>
              {{ isSearchActive ? post.post_content : post.doc.post_content }}
            </p>
            <p>
              Created:
              {{
                new Date(
                  isSearchActive
                    ? post.attributes.creation_date
                    : post.doc.attributes?.creation_date
                ).toLocaleString()
              }}
            </p>
            <button @click="deleteDocument(post._id)" class="delete-btn">
              Delete
            </button>
            <button @click="updateDocument(post._id)" class="update-btn">
              Edit
            </button>
            <!-- Attachments Section -->
            <div
              v-if="post?._attachments && post?._attachments.length > 0"
              class="attachments-section"
            >
              <h4>Attachments:</h4>
              <div
                v-for="attachment in post?._attachments"
                :key="attachment.filename"
                class="attachment-item"
              >
                <a
                  :href="attachment.url"
                  :download="attachment.filename"
                  class="attachment-link"
                >
                  {{ attachment.filename }}
                </a>
              </div>
            </div>

            <!-- Add Attachment Form -->
            <div class="add-attachment-section">
              <input
                type="file"
                @change="handleFileUpload($event)"
                class="file-input"
              />
              <button
                @click="addAttachmentToPost(post._id)"
                class="add-attachment-btn"
                :disabled="!selectedFile"
              >
                Add Attachment
              </button>
            </div>
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
              <button type="submit" class="save-btn">Save</button>
              <button type="button" @click="cancelEdit" class="cancel-btn">
                Cancel
              </button>
            </form>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.searchTitle {
  color: black;
}
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.search-section {
  margin-bottom: 30px;
  padding: 20px;
  background-color: #f5f5f5;
  border-radius: 8px;
}

.search-box {
  margin-bottom: 20px;
}

.search-input {
  width: 100%;
  padding: 8px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.date-range {
  margin-top: 15px;
}

.date-inputs {
  display: flex;
  gap: 15px;
  margin-bottom: 15px;
}

.date-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.search-btn {
  background-color: #4caf50;
  color: white;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.clear-btn {
  background-color: #ff9800;
  color: white;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  margin-top: 15px;
}

.delete-btn {
  background-color: #ff4444;
  color: white;
  padding: 5px 10px;
  margin-right: 5px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.update-btn {
  background-color: #44aaff;
  color: white;
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.create-btn {
  background-color: #4caf50;
  color: white;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.save-btn {
  background-color: #4caf50;
  color: white;
  padding: 5px 10px;
  margin-right: 5px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.cancel-btn {
  background-color: #9e9e9e;
  color: white;
  padding: 5px 10px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.edit-form {
  margin: 1rem 0;
  padding: 1rem;
  background-color: #f5f5f5;
  border-radius: 4px;
}

.no-results {
  text-align: center;
  padding: 20px;
  color: #666;
  font-style: italic;
}

form div {
  margin-bottom: 1rem;
}

label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}

input,
textarea {
  width: 100%;
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

button {
  margin-right: 0.5rem;
}

textarea {
  min-height: 100px;
  resize: vertical;
}

.document-item {
  margin-bottom: 20px;
  padding: 15px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

hr {
  border: none;
  border-top: 1px solid #eee;
  margin: 20px 0;
}
.attachments-section {
    margin: 1rem 0;
    padding: 1rem;
    background-color: #f8f8f8;
    border-radius: 4px;
}

.attachment-item {
    display: flex;
    align-items: center;
    margin: 0.5rem 0;
    padding: 0.5rem;
    background-color: white;
    border-radius: 4px;
}

.attachment-link {
    flex-grow: 1;
    color: #2196f3;
    text-decoration: none;
    margin-right: 1rem;
}

.delete-attachment-btn {
    background-color: #ff4444;
    color: white;
    padding: 4px 8px;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.8rem;
}
</style>

Version 6 of 6
