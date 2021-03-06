<template>
  <div class="mb-0 game-wrapper">
    <v-card 
      class="overflow-hidden"
    >
      <!-- Game Players Drawer -->
      <v-navigation-drawer
          :expand-on-hover="!pane_lock"
      >
      <v-list
        nav
        dense
      >
        <v-list-item two-line >
          <v-list-item-icon>
            <v-icon class="pt-3">
            mdi-account-group
            </v-icon>
          </v-list-item-icon>

          <v-list-item-content>
            <v-list-item-title>Players</v-list-item-title>
          </v-list-item-content>
        </v-list-item>

        <v-divider></v-divider>
            
        <div v-if="gameState.all_players !== undefined">  
          <v-list-item
            v-for="player in gameState.all_players"
            :key="player.name"
            :input-value="player.id === gameState.current_player.id"
            color="#1F7087"
            class="pa-3 player-drawer-item"
            two-line
          >
            <v-list-item-icon>
              <v-icon class="pt-3">
              mdi-account
              </v-icon>
            </v-list-item-icon>
            <v-list-item-content>
              {{ player.name }}
              <ul class="hand ma-0 pa-0">
                <li v-for="(card, index) of player.cards" :key="index">🃏</li>
              </ul>
            </v-list-item-content>
          </v-list-item>
        </div>
        </v-list>
      </v-navigation-drawer>
    </v-card>
    <v-container>
      <v-row >
      <!-- <v-row> -->

        <v-col>
          <!-- Game stats -->
          <v-row>
            <v-card class="ma-3 pa-6" outlined tile>
              <p>
                Current Game id: {{ gameState.game_id }}
              </p>
              <p>
                Status: {{ gameState.status }}
              </p>
              <p>
                Your Name: {{ playerName }}
              </p>
              <p v-if="gameState.draw_pile != undefined">
                Cards Remaining in Draw Pile: {{ gameState.draw_pile.length }}
              </p>
              <p>
                <v-switch v-model="pane_lock" :label="`Unlock Players Pane`"></v-switch>
              </p>
            </v-card>
          </v-row>

          <!-- Current Card and actions -->
          <v-col cols="12" v-if="gameState.status === 'Playing'">
            <v-row v-if="gameState.current_card != undefined">
              <v-card class="center-text ma-3 pa-6" outlined tile>
                <Card
                  :number="gameState.current_card.value"
                  :key="gameState.current_card.color"
                  :color="gameState.current_card.color"
                />
              </v-card>
            </v-row>
          </v-col>
        </v-col>

        <!-- Current cards in the deck -->
        <v-col class="mb-6">
          <v-card
            class="ma-3 pa-6"
            outlined
            tile
          >
            <v-card-text v-if="gameState.status === 'Waiting For Players'">
              <v-row v-if="gameState.creator != undefined && gameState.creator.id == gameState.player_id">
                You are the creator of the game. When you are ready: <v-btn @click.native="startGame">Start Game</v-btn>
              </v-row>
              <v-row v-else>
                Please wait for the creator to start the game.
              </v-row>
            </v-card-text>

            <v-card-text v-if="gameState.status === 'Playing' && gameState.player_id === gameState.current_player.id">
              Click to play a card from your hand or <v-btn @click.native="drawCard">Draw from deck</v-btn>
            </v-card-text>

            <v-card-text v-else-if="gameState.status === 'Playing'">
              Waiting for {{ gameState.current_player.name }}
            </v-card-text>
            
            <v-card-text v-else-if="gameState.status === 'Finished'">
              The game is finished!
            </v-card-text>
          </v-card>

          <div v-if="gameState.status === 'Playing'" >

            <!-- Organize Cards -->
            <v-card :class="'ma-3 pl-6 pa-4'" outlined tile>
              <v-row v-if="loadingHand">
                Loading Original Hand Layout
              </v-row>

              <v-row v-else class="pl-3">
                Organize Cards
                <v-btn @click.native="orgByColor">by Color</v-btn>
                <v-btn @click.native="orgByNum">by Number</v-btn>
                <v-btn @click.native="orgOff">Off</v-btn>
              </v-row>
            </v-card>

            <Card
              v-for="(card, i) in gameState.player_cards"
              :key="i"
              :number="card.value"
              :color="card.color"
              @click.native=" (card.value == 'W' || card.value == 'W4') ? selectWildColor(card) : playCard(card)"
            ></Card>
          </div>
        </v-col>

        <!-- Chat -->
        <v-col v-show="chatOpen" class="float-chat">
          <Chat @snackbarText="runsnackbar" :gameState="gameState"/>
        </v-col>
      </v-row>

    </v-container>

    <div v-if="gameState.status === 'Playing' && gameState.current_player != undefined" @click="chatOpen = !chatOpen" class="float-button">
      Chat
    </div>
    <v-dialog
      v-model="chooseColorDialog.visible"
      persistent
      max-width="500px"
    >
      <v-card >
        <v-card-title
          class="blue"
        >
          Chose color for Wild card
        </v-card-title>
        <v-card-actions>
            <v-col>
              <v-btn
                color="red"
                large
                @click.native="playWildCard('red')"
              >Red</v-btn>
            </v-col>
            <v-col>
              <v-btn
                color="green"
                large
                @click.native="playWildCard('green')"
              >Green</v-btn>
            </v-col>
            <v-col>
              <v-btn
                color="blue"
                large
                @click.native="playWildCard('blue')"
              >Blue</v-btn>
            </v-col>
            <v-col>
              <v-btn
              color="yellow"
              large
                @click.native="playWildCard('yellow')"
              >Yellow</v-btn>
            </v-col>
        </v-card-actions>
      </v-card>
    </v-dialog>

      <v-snackbar
        v-model="snackbar"
        color="info"
        :timeout='4000'
        v-show="gameState.status === 'Playing' && playerName !== newMessageName"
        >{{snackbarText}}
        <v-btn text @click="snackbar=false">
          Close
        </v-btn>
      </v-snackbar>

  </div>
</template>

<script>
import unoService from "../services/unoService";
import Card from "../components/Card";
import Chat from "../components/Chat";

export default {
  name: "Game",
  components: {
    Card,
    Chat,
  },
  data() {
    return {
      pane_lock: true,
      gameState: {},
      cards: [],

      chatOpen: false,
      
      playerName: "",
      chooseColorDialog: {
        visible: false,
        card: {},
        color: ""
      },

      sortByNum: false,
      sortByColor: false,
      loadingHand: false,
      colors: { 'red': 0, 'blue': 1 , 'green': 2, 'yellow': 3, 'wild': 4},
      values: { '1' : 0, '2' : 1, '3' : 2, '4' : 3, '5' : 4, '6' : 5, '7' : 6, '8' : 7, '9' : 8, 'S' : 9, 'R' : 10, 'W' : 11, 'D2' : 12, 'W4' : 13},
    
      snackbar: false,
      snackbarText: "",
      newMessageName: "",
    };
  },

  methods: {
    async updateData() {
      let res = await unoService.getGameState(this.$route.params.id);

      if (res.data != null) {
        this.gameState = res.data;
      }
      this.decideSort()
    },
    decideSort() {
      if (this.sortByColor) {
        this.orgByColor()
      }else if (this.sortByNum) {
        this.orgByNum()
      }else{
        this.loadingHand = false
      }
    },
    orgOff() {
      if (this.sortByColor == true || this.sortByNum == true) {
        this.loadingHand = true;
      }
      this.sortByNum = false;
      this.sortByColor = false;
    },
    orgByNum() {
      if (this.gameState.player_cards != undefined) {
        this.gameState.player_cards.sort((a, b) => (this.colors[a.color] > this.colors[b.color]) ? 1 : -1 );
        this.gameState.player_cards.sort((a, b) => (this.values[a.value] > this.values[b.value]) ? 1 : -1 );
      }

      this.sortByNum = true;
      this.sortByColor = false;
    },
    orgByColor() {
      if (this.gameState.player_cards != undefined) {
        this.gameState.player_cards.sort((a, b) => (this.values[a.value] < this.values[b.value]) ? 1 : -1 );
        this.gameState.player_cards.sort((a, b) => (this.colors[a.color] < this.colors[b.color]) ? 1 : -1 );
      }

      this.sortByNum = false;
      this.sortByColor = true;
    },
    async startGame() {
      await unoService.startGame(this.$route.params.id);
      // TODO make sure startGame endpoint returns the game state and then remove this call to updateData()
      this.updateData(); 
    },

    runsnackbar(name, message) {
      this.newMessageName = name;
      this.snackbarText = name + " says: " + message;
      this.snackbar = true;
    },

    selectWildColor(card)
    {
      this.chooseColorDialog.card = card;
      this.chooseColorDialog.visible = true;
    },

    async playWildCard(color) {
      this.chooseColorDialog.visible = false;
      this.chooseColorDialog.card.color = color;
      this.playCard(this.chooseColorDialog.card);
    },

    async playCard(card) { 
      console.log("Playing card", card);     
      let res = await unoService.playCard(this.$route.params.id, card.value, card.color);
      
      if (res.data) {
        this.gameState = res.data;
      }
    },
    sendMessage() {
      this.snackbarText = this.username + " says: " + this.newMessage;
      this.snackbar = true;
    },
    async drawCard() {
      let res = await unoService.drawCard(this.$route.params.id);
      
      if (res.data) {
        this.gameState = res.data;
      }
    }
  },
  created() {
    this.updateData();
    this.updateInterval = setInterval(() => {
      this.updateData();
    }, 2000);
  },
  mounted() {
    unoService.getPlayerNameFromToken()
    .then( resp => {
        this.playerName = resp?.data?.name
    })
    .catch(err => {
      console.err("Could not get player name from assigned token\n", err)
    })
  },
  beforeDestroy (){
    if(this.updateInterval){
      clearInterval(this.updateInterval);
    }
  },
};
</script>

<style scoped>

.float-chat {
  position:fixed;
	width:450px;
	height:550px;
  bottom:125px;
  right:50px;
	background-color:#00263A;
	color:#FFF;
	border-radius:10px;
  /* border-width: 5px;
  border-color: #FFF;
  outline: #FFF; */
  padding: 5px 5px 5px 5px;
}

.float-button{
	position:fixed;
	width:60px;
	height:60px;
	bottom:50px;
	right:50px;
	background-color:#00263A;
	color:#FFF;
	border-radius:50px;
  text-align:center;
  padding: 20px 0px 0px 0px;
	/* box-shadow: 2px 2px 3px #999; */
}

.game-wrapper {
  display: flex; 
  min-height: 100%;
  background-color: black;
}

.player-drawer-item {
  overflow: hidden;
}

.player-drawer-item > div {
  overflow: hidden;
}

.hand {
  overflow: hidden;
}

.v-btn {
  margin: 0px 10px 0px 10px;
}

.hand > li {
  display: inline;
}

.hand span {
  font-weight: bold;
}

</style>
