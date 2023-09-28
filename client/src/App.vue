<script>
  import {io} from 'socket.io-client'
  const socket = io('localhost:3010');

  export default {
    name: 'App',
    components: {
    },
    data() {
      return {
        view: "select",
        singleplayer: false,
         
        content: [
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
          ["", "", "", "", "", "", "", "", ""],
        ],
        game: ["", "", "", "", "", "", "", "", ""],

        fieldToPlay: -1, // -1 = free, 0-8 = field


        turn: true,
        isOver: false,
        winner: null,
        isTie: false,

        yourTurn: true,

        lobbyfull: false,

        room: "",
      }
    },
    methods: {
      draw(index, field, drawFromOther = false) {
        if(this.singleplayer){
          if(this.content[field][index] != "" || this.isOver){
            return;
          }

          if(this.fieldToPlay != field && this.fieldToPlay != -1){
            return;
          }

          this.turn ? this.content[field][index] = "X" : this.content[field][index] = "O";

          this.turn = !this.turn;
          // calculate the winner
          this.fieldToPlay = index;
          this.calculateWinner(field);
          this.calculateTie(field)
        } else {
          if(this.content[field][index] != "" || this.isOver || !this.yourTurn){
            return;
          }

          this.turn ? this.content[field][index] = "X" : this.content[field][index] = "O";

          if (!drawFromOther) {
            socket.emit("play", index);
            this.yourTurn = false;
          } 


          this.turn = !this.turn;
          // calculate the winner
          this.calculateWinner(field);
          this.calculateTie(field)
        }
        
      },


      calculateWinner(field) {
        const WIN_CONDITIONS = [
          // rows
          [0, 1, 2], [3, 4, 5], [6, 7, 8],
          // cols
          [0, 3, 6], [1, 4, 7], [2, 5, 8],
          // diagonals
          [0, 4, 8], [2, 4, 6]
        ];
        for (let i = 0; i < WIN_CONDITIONS.length; i++) {
          let firstIndex = WIN_CONDITIONS[i][0];
          let secondIndex = WIN_CONDITIONS[i][1];
          let thirdIndex = WIN_CONDITIONS[i][2];
          if(this.content[field][firstIndex] == this.content[field][secondIndex] &&
                  this.content[field][firstIndex] == this.content[field][thirdIndex] &&
                  this.content[field][firstIndex] != "") {
            this.game[field] = this.content[field][firstIndex];
            this.fieldToPlay = -1;
            this.calculateGameWinner();
            this.calculateGameTie();
            //reset visual restrictions
          }
        }
      },


      calculateTie(_field){
        for (let i = 0 ; i<= 8 ; i++) {
          if(this.content[_field][i] == "") {
            return;
          }
        }
        this.isTie = true;
      },

      calculateGameWinner(){
        const WIN_CONDITIONS = [
          // rows
          [0, 1, 2], [3, 4, 5], [6, 7, 8],
          // cols
          [0, 3, 6], [1, 4, 7], [2, 5, 8],
          // diagonals
          [0, 4, 8], [2, 4, 6]
        ];
        for (let i = 0; i < WIN_CONDITIONS.length; i++) {
          let firstIndex = WIN_CONDITIONS[i][0];
          let secondIndex = WIN_CONDITIONS[i][1];
          let thirdIndex = WIN_CONDITIONS[i][2];
          if(this.game[firstIndex] == this.game[secondIndex] &&
                  this.game[firstIndex] == this.game[thirdIndex] &&
                  this.game[firstIndex] != "") {
            this.isOver = true;
            this.winner = this.game[firstIndex];
          }
        }
      },

      calculateGameTie(){
        for (let i = 0 ; i<= 8 ; i++) {
          if(this.games[i] == "") {
            return;
          }
        }
        this.isTie = true;
      },

      
      resetBoard(_self) {
        if(_self) {socket.emit("reset")}
        for (let i=0; i < 9; i++) {
          for(let j=0; j < 9; j++){
            this.content[j][i] = "";
          }
          this.game[i] = "";
        }
        this.isOver = false;
        this.winner = null
        this.isTie = false
      },


      HostRoom(){
        socket.emit("host");
        this.view = 'game';
      },

      JoinRoom(){
        socket.emit("joinGame", this.room);
        this.lobbyfull = true;
      },

      CopyToClipboard(txt){
        navigator.clipboard.writeText(txt);
      }
    },

    created(){
      socket.on("test", (msg) => {
        console.log("received msg from server", msg)
      });

      socket.on("play", (index) => {
        console.log("received index", index)
        this.yourTurn = true;
        this.draw(index, true)
      });

      socket.on("reset", () => {
        console.log("received index", "reset")
        this.resetBoard(false);
      })

      socket.on("cantJoinFull", () => {
        alert("cant join Room (Room full)");
      });

      socket.on("cantJoinNotFound", () => {
        alert("cant join Room (Room not Found)");
      });

      socket.on('joined', () => {
        this.view = 'game';
      });

      socket.on('createdRoom', (_room) => {
       this.room = _room;
      });

      socket.on('userJoined', () => {
        this.resetBoard();
        this.lobbyfull = true;
      });

      socket.on('opponentLeft', () => {
        this.resetBoard();
        this.lobbyfull = false;
      });

    },
  }
</script>

<template>

  <div id="select" v-if="view == 'select'">
    <div id="selectWrapper">
      <div class="selectField">
        <button @click="singleplayer = true; view = 'game'">SinglePlayer</button>
      </div>
      <div class="selectField">
        <button @click="HostRoom()">Host Room</button>
      </div>
      <div class="selectField">
        <input type="text" placeholder="room Id" v-model="room"><br>
        <button @click="JoinRoom()">join Room</button>
      </div>

    </div>
  </div>

  <div id="game" v-else>
    <p style="margin: 15px; position: absolute;" class="clickable" v-if="!singleplayer" @click="CopyToClipboard(room)">Room:{{ room }}</p>
    <p style="margin: 15px; position: absolute; right: 0;" v-if="!singleplayer && !lobbyfull">currently waiting for opponent</p>

    <div class="container unmarkable">
      <h1>Tic-Tac-Toe</h1>
      <div class="play-area">
        <div class="wonboard" v-if="game[0] != ''">{{ game[0] }}</div>
        <div class="smallboard" id="sb0" v-else> <!-- board 0 -->
          <div class="block_0 block" @click="draw(0, 0)">{{ content[0][0] }}</div>
          <div class="block_1 block" @click="draw(1, 0)">{{ content[0][1] }}</div>
          <div class="block_2 block" @click="draw(2, 0)">{{ content[0][2] }}</div>
          <div class="block_3 block" @click="draw(3, 0)">{{ content[0][3] }}</div>
          <div class="block_4 block" @click="draw(4, 0)">{{ content[0][4] }}</div>
          <div class="block_5 block" @click="draw(5, 0)">{{ content[0][5] }}</div>
          <div class="block_6 block" @click="draw(6, 0)">{{ content[0][6] }}</div>
          <div class="block_7 block" @click="draw(7, 0)">{{ content[0][7] }}</div>
          <div class="block_8 block" @click="draw(8, 0)">{{ content[0][8] }}</div> 
        </div>

        <div class="wonboard" v-if="game[1] != ''">{{ game[1] }}</div>
        <div class="smallboard" id="sb1" v-else> <!-- board 1 -->
          <div class="block_0 block" @click="draw(0, 1)">{{ content[1][0] }}</div>
          <div class="block_1 block" @click="draw(1, 1)">{{ content[1][1] }}</div>
          <div class="block_2 block" @click="draw(2, 1)">{{ content[1][2] }}</div>
          <div class="block_3 block" @click="draw(3, 1)">{{ content[1][3] }}</div>
          <div class="block_4 block" @click="draw(4, 1)">{{ content[1][4] }}</div>
          <div class="block_5 block" @click="draw(5, 1)">{{ content[1][5] }}</div>
          <div class="block_6 block" @click="draw(6, 1)">{{ content[1][6] }}</div>
          <div class="block_7 block" @click="draw(7, 1)">{{ content[1][7] }}</div>
          <div class="block_8 block" @click="draw(8, 1)">{{ content[1][8] }}</div> 
        </div>        

        <div class="wonboard" v-if="game[2] != ''">{{ game[2] }}</div>
        <div class="smallboard" id="sb2" v-else>  <!-- board 2 -->
          <div class="block_0 block" @click="draw(0, 2)">{{ content[2][0] }}</div>
          <div class="block_1 block" @click="draw(1, 2)">{{ content[2][1] }}</div>
          <div class="block_2 block" @click="draw(2, 2)">{{ content[2][2] }}</div>
          <div class="block_3 block" @click="draw(3, 2)">{{ content[2][3] }}</div>
          <div class="block_4 block" @click="draw(4, 2)">{{ content[2][4] }}</div>
          <div class="block_5 block" @click="draw(5, 2)">{{ content[2][5] }}</div>
          <div class="block_6 block" @click="draw(6, 2)">{{ content[2][6] }}</div>
          <div class="block_7 block" @click="draw(7, 2)">{{ content[2][7] }}</div>
          <div class="block_8 block" @click="draw(8, 2)">{{ content[2][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[3] != ''">{{ game[3] }}</div>
        <div class="smallboard" id="sb3" v-else>  <!-- board 3 -->
          <div class="block_0 block" @click="draw(0, 3)">{{ content[3][0] }}</div>
          <div class="block_1 block" @click="draw(1, 3)">{{ content[3][1] }}</div>
          <div class="block_2 block" @click="draw(2, 3)">{{ content[3][2] }}</div>
          <div class="block_3 block" @click="draw(3, 3)">{{ content[3][3] }}</div>
          <div class="block_4 block" @click="draw(4, 3)">{{ content[3][4] }}</div>
          <div class="block_5 block" @click="draw(5, 3)">{{ content[3][5] }}</div>
          <div class="block_6 block" @click="draw(6, 3)">{{ content[3][6] }}</div>
          <div class="block_7 block" @click="draw(7, 3)">{{ content[3][7] }}</div>
          <div class="block_8 block" @click="draw(8, 3)">{{ content[3][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[4] != ''">{{ game[4] }}</div>
        <div class="smallboard" id="sb4" v-else>  <!-- board 4 -->
          <div class="block_0 block" @click="draw(0, 4)">{{ content[4][0] }}</div>
          <div class="block_1 block" @click="draw(1, 4)">{{ content[4][1] }}</div>
          <div class="block_2 block" @click="draw(2, 4)">{{ content[4][2] }}</div>
          <div class="block_3 block" @click="draw(3, 4)">{{ content[4][3] }}</div>
          <div class="block_4 block" @click="draw(4, 4)">{{ content[4][4] }}</div>
          <div class="block_5 block" @click="draw(5, 4)">{{ content[4][5] }}</div>
          <div class="block_6 block" @click="draw(6, 4)">{{ content[4][6] }}</div>
          <div class="block_7 block" @click="draw(7, 4)">{{ content[4][7] }}</div>
          <div class="block_8 block" @click="draw(8, 4)">{{ content[4][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[5] != ''">{{ game[5] }}</div>
        <div class="smallboard" id="sb5" v-else>  <!-- board 5 -->
          <div class="block_0 block" @click="draw(0, 5)">{{ content[5][0] }}</div>
          <div class="block_1 block" @click="draw(1, 5)">{{ content[5][1] }}</div>
          <div class="block_2 block" @click="draw(2, 5)">{{ content[5][2] }}</div>
          <div class="block_3 block" @click="draw(3, 5)">{{ content[5][3] }}</div>
          <div class="block_4 block" @click="draw(4, 5)">{{ content[5][4] }}</div>
          <div class="block_5 block" @click="draw(5, 5)">{{ content[5][5] }}</div>
          <div class="block_6 block" @click="draw(6, 5)">{{ content[5][6] }}</div>
          <div class="block_7 block" @click="draw(7, 5)">{{ content[5][7] }}</div>
          <div class="block_8 block" @click="draw(8, 5)">{{ content[5][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[6] != ''">{{ game[6] }}</div>
        <div class="smallboard" id="sb6" v-else>  <!-- board 6 -->
          <div class="block_0 block" @click="draw(0, 6)">{{ content[6][0] }}</div>
          <div class="block_1 block" @click="draw(1, 6)">{{ content[6][1] }}</div>
          <div class="block_2 block" @click="draw(2, 6)">{{ content[6][2] }}</div>
          <div class="block_3 block" @click="draw(3, 6)">{{ content[6][3] }}</div>
          <div class="block_4 block" @click="draw(4, 6)">{{ content[6][4] }}</div>
          <div class="block_5 block" @click="draw(5, 6)">{{ content[6][5] }}</div>
          <div class="block_6 block" @click="draw(6, 6)">{{ content[6][6] }}</div>
          <div class="block_7 block" @click="draw(7, 6)">{{ content[6][7] }}</div>
          <div class="block_8 block" @click="draw(8, 6)">{{ content[6][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[7] != ''">{{ game[7] }}</div>
        <div class="smallboard" id="sb7" v-else>  <!-- board 7 -->
          <div class="block_0 block" @click="draw(0, 7)">{{ content[7][0] }}</div>
          <div class="block_1 block" @click="draw(1, 7)">{{ content[7][1] }}</div>
          <div class="block_2 block" @click="draw(2, 7)">{{ content[7][2] }}</div>
          <div class="block_3 block" @click="draw(3, 7)">{{ content[7][3] }}</div>
          <div class="block_4 block" @click="draw(4, 7)">{{ content[7][4] }}</div>
          <div class="block_5 block" @click="draw(5, 7)">{{ content[7][5] }}</div>
          <div class="block_6 block" @click="draw(6, 7)">{{ content[7][6] }}</div>
          <div class="block_7 block" @click="draw(7, 7)">{{ content[7][7] }}</div>
          <div class="block_8 block" @click="draw(8, 7)">{{ content[7][8] }}</div>           
        </div>

        <div class="wonboard" v-if="game[8] != ''">{{ game[8] }}</div>
        <div class="smallboard" id="sb8" v-else>  <!-- board 8 -->
          <div class="block_0 block" @click="draw(0, 8)">{{ content[8][0] }}</div>
          <div class="block_1 block" @click="draw(1, 8)">{{ content[8][1] }}</div>
          <div class="block_2 block" @click="draw(2, 8)">{{ content[8][2] }}</div>
          <div class="block_3 block" @click="draw(3, 8)">{{ content[8][3] }}</div>
          <div class="block_4 block" @click="draw(4, 8)">{{ content[8][4] }}</div>
          <div class="block_5 block" @click="draw(5, 8)">{{ content[8][5] }}</div>
          <div class="block_6 block" @click="draw(6, 8)">{{ content[8][6] }}</div>
          <div class="block_7 block" @click="draw(7, 8)">{{ content[8][7] }}</div>
          <div class="block_8 block" @click="draw(8, 8)">{{ content[8][8] }}</div>           
        </div>


      
      </div>

      <h2 id="winner" v-if="isOver"> Winner is {{winner}} </h2>
      <h2 v-if="isTie"> Game is Tie</h2>

      <button @click="resetBoard(true); " v-if="isOver || isTie">RESET BOARD</button>
    </div>
  </div>
  
</template>


<style>
  * {
    color: white;

    margin: 0;
    padding: 0;
    box-sizing: border-box;
    font-family: Arial, Helvetica, sans-serif;
  }
  .container {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
  h1 {
    font-size: 5rem;
    margin-bottom: 0.5em;
  }
  h2 {
    margin-top: 1em;
    font-size: 2rem;
    margin-bottom: 0.5em;
  }
  .play-area {
    display: grid;
    width: 700px;
    height: 700px;
    grid-template-columns: auto auto auto;
  }
  .smallboard{
    display: flex;
    width: 233.33px;
    height: 233.33px;
    align-items: center;
    justify-content: center;
    flex-wrap: wrap;
    font-size: 3rem;
    font-weight: bold;
    /* border: 4px solid rgb(255, 0, 157); */
    transition: background 0.2s ease-in-out;
    padding-top: 10px;
    padding-bottom: 10px;
  }

  #sb0,#sb1,#sb2,#sb3,#sb4,#sb5{
    border-bottom: 3px solid pink;
  } 

  #sb0,#sb3,#sb6,#sb1,#sb4,#sb7{
    border-right: 3px solid pink;
  }

  #sb6,#sb7,#sb8,#sb3,#sb4,#sb5{
    border-top: 3px solid pink;
  } 

  #sb2,#sb5,#sb8,#sb1,#sb4,#sb7{
    border-left: 3px solid pink;
  }

 

  .block {
    display: flex;
    width: 70px;
    height:70px;
    align-items: center;
    justify-content: center;
    font-size: 3rem;
    font-weight: bold;
    border: 2px solid rgb(255, 255, 255);
    transition: background 0.2s ease-in-out;
  }
  .block:hover {
    cursor: pointer;
    background: #0ff30f;
  }
  .occupied:hover {
    background: #ff3a3a;
  }
  .win {
    background: #0ff30f;
  }
  .win:hover {
    background: #0ff30f;
  }
  .block_0,
  .block_1,
  .block_2 {
    border-top: none;
  }
  .block_0,
  .block_3,
  .block_6 {
    border-left: none;
  }
  .block_6,
  .block_7,
  .block_8 {
    border-bottom: none;
  }
  .block_2,
  .block_5,
  .block_8 {
    border-right: none;
  }
  #game button {
    outline: none;
    border: 4px solid green;
    padding: 10px 20px;
    font-size: 1rem;
    font-weight: bold;
    background: none;
    transition: all 0.2s ease-in-out;
  }
  button:hover {
    cursor: pointer;
    background: green;
    color: white;
  }
  .playerWin {
    color: green;
  }
  .computerWin {
    color: red;
  }
  .draw {
    color: orangered;
  }


  /* select */
  #select{
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);

    text-align: center;
  }

  .selectField{
    margin: 20px;
  }

  .selectField button{
    background-color: transparent;
    border: 3px solid white;
    padding: 5px;
    width: 120px;
  }

  .selectField input{
    margin-top: 10px;
    background-color: transparent; 
    border: none; 
    border-bottom: 1px solid white; 
    text-align: center;
    margin-bottom: 5px;
  }






  .unmarkable{
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -khtml-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }

  .clickable{
    cursor: pointer;
  }

  @media only screen and (max-width: 600px) {
    h1 {
      font-size: 3rem;
      margin-bottom: 0.5em;
    }
    h2 {
      margin-top: 1em;
      font-size: 1.3rem;
    }
  }
</style>