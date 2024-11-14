<script lang="ts">
import { ref } from "vue";
import PouchDB from "pouchdb";

declare interface Post {
  _id: string;
  doc: {
    post_name: string;
    post_content: string;
    attributes: {
      creation_date: string;
    };
  };
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      newDoc: {
        title: "",
        content: "",
        timestamp: "",
      },
    };
  },

  mounted() {
    this.initDatabase();
    this.fetchData();
  },

  methods: {
    putDocument(document: Post) {
      const db = ref(this.storage).value;
      if (db) {
        db.put(document)
          .then(() => {
            console.log("Add ok");
          })
          .catch((error) => {
            console.log("Add ko", error);
          });
      }
    },

    fetchData() {
      const storage = ref(this.storage);
      const self = this;
      if (storage.value) {
        console.log(storage.value);
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true,
          })
          .then(
            function (result: any) {
              console.log("fetchData success", result);
              self.postsData = result.rows;
            }.bind(this)
          )
          .catch(function (error: any) {
            console.log("fetchData error", error);
          });
      }
    },
    async createDocument() {
      const db = ref(this.storage).value;
      //Preparing the document
      const doc = {
        // Spread operator to include all newDoc properties
        ...this.newDoc,
        // Add creation timestamp
        timestamp: new Date().toISOString(),
      };
      //Inserting Document
      try {
        if (db != null) {
          const response = await db.post(doc);
          if (response.ok) {
            return console.log("oui");
            this.newDoc = {
            title: '',
            content: '',
            timestamp: ''
          };
          } else {
            console.log("error");
          }
        }
      } catch (err) {
        console.log(err);
      }
    },
    deleteDocument(){
      const db = ref(this.storage).value;
      
    },
    initDatabase() {
      const db = new PouchDB("http://admin:couchdb@localhost:5984/post");
      if (db) {
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
  <h1>Nombre de post: {{ postsData.length }}</h1>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.doc.post_name
        }}<em
          style="font-size: x-small"
          v-if="post.doc.attributes?.creation_date"
        >
          - {{ post.doc.attributes?.creation_date }}
        </em>
      </div>
    </li>
  </ul>
  <div>
    <form @submit.prevent="createDocument">
      <div>
        <label for="title">Title:</label>
        <input id="title" v-model="newDoc.title" required />
      </div>
      <div>
        <label for="content">Content:</label>
        <textarea id="content" v-model="newDoc.content" required></textarea>
      </div>
      <button type="submit">Create Document</button>
    </form>
  </div>
</template>
