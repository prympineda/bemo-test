<template>
  <div class="content">
    <div class="container">

      <div class="columns"
          v-for="(column, columnIndex) in columns"
          :key="columnIndex">
        <div>
          <button class="delete-column-btn" @click="deleteConfirmation(columnIndex)" :disabled="buttonDisabled">X</button>
          <h3 class="column-title">{{column.name}}</h3>
          <draggable
            :list="column.cards"
            group="tasks"
            @start="drag=true" @end="drag=false"
            :move="onMoveCallback"
            :data-column-key="columnIndex"
            :disabled="cardDisabled"
          >
            <div
              class="cards"
              v-for="(card, cardIndex) in column.cards"
              :key="cardIndex"
              @click="showCard(columnIndex, cardIndex, card.id, card.name, card.description)"
              @drop="movedOrder()"
             
            >
              {{ card.name }}
            </div>

            <div>
              <button @click="addCard(columnIndex)" :disabled="buttonDisabled">Add Card</button>
            </div>
          </draggable>
        </div>
      </div>

       <div class="columns">
        <div>
          <h3 class="column-title"><button @click="showModal" :disabled="buttonDisabled">Add</button> </h3>
        </div>
      </div>

      <modal name="api-error-modal" :clickToClose="false">
        <h1>An error occured. Please refresh the page.</h1>
      </modal>

      <modal name="addColumnModal">
          <div>
              <label>Column Name</label>
              <input type="text" v-model="newColumn">

              <button @click="addColumn" :disabled="buttonDisabled">Add Column</button>
          </div>
      </modal>
      
      <modal name="addCardModal">
          <div>
              <label for="update-card-name">Card Name</label>
              <input id="update-card-name" type="text" v-model="selectedCard.name">
          </div>
          <div>
              <label for="update-card-descpription">Card Description</label>
              <textarea id="update-card-descpription" cols="30" rows="10" v-model="selectedCard.description"></textarea>
          </div>
          <button @click="saveCard" :disabled="buttonDisabled">Save Card</button>
      </modal>

      <modal name="showCardModal">
          <div>
              <label for="update-card-name">Card Name</label>
              <input id="update-card-name" type="text" v-model="selectedCard.name">
          </div>
          <div>
              <label for="update-card-descpription">Card Description</label>
              <textarea id="update-card-descpription" cols="30" rows="10" v-model="selectedCard.description"></textarea>
          </div>
          <button @click="updateCard" :disabled="buttonDisabled">Update Card</button>
      </modal>

      <modal name="deleteConfirmationModal">
          <div>
              Are you sure you want to delete this column?
          </div>
          <button @click="$modal.hide('deleteConfirmationModal')" >Cancel</button>
          <button @click="removeColumn" :disabled="buttonDisabled">Confirm</button>
      </modal>
    </div>
  </div>
</template>

<script>
import draggable from "vuedraggable";
import axios from 'axios';
const API = process.env.VUE_APP_API;

const axios_header = {
  headers: {
      'Authorization': 'Bearer ' + process.env.VUE_APP_API_KEY,
      'Accept': 'application/json'
  }
}

export default {
  name: "kanban-board",
  components: {
    //import draggable as a component
    draggable
  },
  data() {
    return {
      // for new tasks
      newColumn: "",
      columns: null,
      selectedCard: {
        "id": 0,
        "name": "",
        "description": ""
      },
      selectedCardKey: null,
      selectedColumnKey: null,
      move: {
        "column_id": null,
        "card_id": null,
        "order": null,
      },
      cardDisabled: false,
      buttonDisabled: true,
      
    };
  },
  methods: {
    //add new tasks method
    addColumn() {
      if (this.newColumn) {
        this.buttonDisabled = true;
        axios.post(API+'/columns/store', {'name': this.newColumn}, axios_header)
            .then((response) => {
              let data = response.data
              if (data.success)
              {
                // this.columns = data.data
                this.columns.push(
                { 
                  "name": data.data.name,
                  "id": data.data.id,
                  "cards": data.data.cards
                });

                this.newColumn = "";
                this.$modal.hide('addColumnModal');
              }
            })
            .catch((e) => {
                console.log(e);
                this.buttonDisabled = false
            })
            .finally(() => {
                this.buttonDisabled = false
            })
      }
    },
    addCard (columnKey) {
        this.selectedColumnKey = columnKey
        this.$modal.show('addCardModal')
    },
    showModal () {
        this.$modal.show('addColumnModal')
    },
    hide () {
        this.$modal.hide('addColumnModal');
    },
    onMoveCallback: function(event)
    {
      this.move.column_id = this.columns[event.to.getAttribute('data-column-key')].id
      this.move.card_id = event.draggedContext.element.id
      this.move.order = event.draggedContext.futureIndex + 1
    },
    movedOrder ()
    {
      this.cardDisabled = true
      axios.post(API+'/cards/update-order', 
        this.move, 
        axios_header)
        .then(() => {
          // let data = response.data
          // if (data.success)
          // {
            
          //   // this.newColumn = "";
          //   // this.$modal.hide('addCardModal');

          // }
        })
        .catch((e) => {
            console.log(e);
        })
        .finally(() => {
          this.cardDisabled = false
        })
    },
    showCard (columnKey, cardKey, cardId, name, description)
    {
      this.selectedColumnKey = columnKey
      this.selectedCardKey = cardKey
      this.selectedCard.id = cardId
      this.selectedCard.name = name
      this.selectedCard.description = description
      this.$modal.show('showCardModal')
    },
    saveCard ()
    {
      this.buttonDisabled = true
      axios.post(API+'/cards/store', 
        {
          'column_id': this.columns[this.selectedColumnKey].id,
          'name': this.selectedCard.name,
          'description': this.selectedCard.description 
        }, 
        axios_header)
        .then((response) => {
          let data = response.data
          if (data.success)
          {
            this.columns[this.selectedColumnKey].cards.push(
            { 
              "id": data.data.id,
              "name": data.data.name,
              "description": data.data.description,
            });

            this.newColumn = "";
            this.$modal.hide('addCardModal');
          }
        })
        .catch((e) => {
            console.log(e)
            this.buttonDisabled = false
        })
        .finally(() => {
            this.buttonDisabled = false
        })
    },
    updateCard ()
    {
      this.buttonDisabled = true
      axios.post(API+'/cards/update/'+this.selectedCard.id, 
        {
          'name': this.selectedCard.name,
          'description': this.selectedCard.description 
        }, 
        axios_header)
        .then((response) => {
          let data = response.data
          if (data.success)
          {
            this.columns[this.selectedColumnKey].cards[this.selectedCardKey].name = data.data.name
            this.columns[this.selectedColumnKey].cards[this.selectedCardKey].description = data.data.description
            this.selectedCard.name = "";
            this.selectedCard.description = "";
            this.$modal.hide('showCardModal');
          }
        })
        .catch((e) => {
            console.log(e);
            this.buttonDisabled = false
        })
        .finally(() => {
            this.buttonDisabled = false
        })
    },
    deleteConfirmation (columnKey)
    {
      this.selectedColumnKey = columnKey
      this.$modal.show('deleteConfirmationModal')
    },
    removeColumn ()
    {
      this.buttonDisabled = true
      axios.post(API+'/columns/delete', 
        {
          'column_id': this.columns[this.selectedColumnKey].id
        }, 
        axios_header)
        .then((response) => {
          let data = response.data
          if (data.success)
          {
            this.columns.splice(this.selectedColumnKey, 1);
          }
        })
        .catch((e) => {
            console.log(e);
        })
        .finally(() => {
            this.$modal.hide('deleteConfirmationModal')
            this.buttonDisabled = false
        })
    }
  },
  created() {
    axios.get(API+'/columns', axios_header)
      .then((response) => {
        let data = response.data
        if (data.success)
        {
          this.columns = data.data
          this.buttonDisabled = false
        }
      })
      .catch(() => {
        this.$modal.show('api-error-modal')
      })
   }
};
</script>

<style lang="scss">
/* light stylings for the kanban columns */
// .kanban-column {
//   min-height: 300px;
// }

$base-color: hsl(336, 33%, 3%);
$border-color: rgba($base-color, 0.88);

.content {
  overflow-x: scroll;
}

.container {
  display: flex;
}

.columns {
  min-width: 250px;
  border: 1px solid $border-color;
  height: fit-content;
  margin: 5px;
}

.column-title {
  font-weight: 1000;
  text-align: center;
  color: $base-color;
  border-bottom: 1px solid $border-color;
  padding: 10px;
  margin: 10px;
}

.delete-column-btn {
  float: right;
}

.cards {
   border: 1px solid $border-color;
   padding: 10px;
   margin: 10px 5px;
}

input[type=text] {
  width: 100%;
  padding: 12px 20px;
  margin: 8px 0;
  box-sizing: border-box;
}

</style>
