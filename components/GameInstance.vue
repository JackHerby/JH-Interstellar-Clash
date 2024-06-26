<script lang="ts" setup>
import { useDisplay } from 'vuetify'
import {
  collection,
  query,
  addDoc,
  updateDoc,
  where,
  doc,
  arrayUnion,
  getDocs,
} from 'firebase/firestore'
import { EGameMode } from '@/enums/EGameMode'
import type { ICard } from '@/interfaces/ICard'
import type { IHighScores, IGame } from '@/interfaces/IHighScores'

const { smAndDown } = useDisplay()

const db = useFirestore()

const peopleRef = collection(db, 'people')
const starshipsRef = collection(db, 'starships')
const peopleDeck = useCollection<ICard>(query(peopleRef))
const starshipsDeck = useCollection<ICard>(query(starshipsRef))

const playerOne = usePlayerOne()
const playerTwo = usePlayerTwo()
const deckType = useDeckType()
const gameMode = useGameMode()
const isGameRunning = useGameRunning()

const hasPlayerOneWon = ref(false)
const playerOneScore = ref(0)
const playerTwoScore = ref(0)

const isActive = ref(false)
const showScore = ref(false)

let playerOneCard = reactive<ICard>({
  name: '',
  description: '',
  weapon: '',
  resource: {
    type: '',
    value: 0,
  },
})
let playerTwoCard = reactive<ICard>({
  name: '',
  description: '',
  weapon: '',
  resource: {
    type: '',
    value: 0,
  },
})

const getDeck = computed(() => {
  if (!peopleDeck.value || !starshipsDeck.value) {
    return
  }
  return deckType.value === 'people' ? peopleDeck.value : starshipsDeck.value
})

// Fisher–Yates shuffle algorithm
const shuffleDeck = (deckToShuffle: ICard[]): ICard[] => {
  for (let i = deckToShuffle.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    const temp = deckToShuffle[i]
    deckToShuffle[i] = deckToShuffle[j]
    deckToShuffle[j] = temp
  }
  return deckToShuffle
}

const saveUserScore = async (): Promise<void> => {
  const user = useCurrentUser()
  if (!user.value) {
    return
  }

  const userId = user.value.uid
  const game: IGame = {
    vs: playerTwo.value,
    userScore: playerOneScore.value,
    oponentScore: playerTwoScore.value,
  }
  const highScores: IHighScores = { userId, games: arrayUnion(game) as unknown as IGame[] }

  const userHighScoresRef = collection(db, 'highScores')
  const userHighScoresQuery = query(userHighScoresRef, where('userId', '==', user.value?.uid))
  const userHighScoresQuerySnapshot = await getDocs(userHighScoresQuery)

  if (userHighScoresQuerySnapshot.empty) {
    addDoc(userHighScoresRef, highScores)
  } else {
    const userHighScoresId = userHighScoresQuerySnapshot.docs[0].id
    const docToUpdate = doc(db, 'highScores', userHighScoresId)
    updateDoc(docToUpdate, { games: arrayUnion(game) })
  }
}

const playGame = (deck: ICard[] | undefined): void => {
  if (!deck) {
    return
  }
  isActive.value = true
  const shuffledDeck = shuffleDeck(deck)
  playerOneCard = shuffledDeck[0]
  playerTwoCard = shuffledDeck[1]
  hasPlayerOneWon.value = playerOneCard.resource.value > playerTwoCard.resource.value
  hasPlayerOneWon.value === true ? playerOneScore.value++ : playerTwoScore.value++
}

const replay = () => {
  isActive.value = false
  showScore.value = false
}

const goToMainMenu = () => {
  saveUserScore()
  isActive.value = false
  showScore.value = false
  playerOneScore.value = 0
  playerTwoScore.value = 0
  playerOne.value = ''
  playerTwo.value = ''
  isGameRunning.value = false
}

onMounted(() => {
  if (gameMode.value === EGameMode.onePlayer) {
    playerTwo.value = 'Computer'
  }
})
</script>

<template>
  <div>
    <h4 v-if="gameMode === EGameMode.onePlayer" class="text-h4 mb-6">Player vs. Computer</h4>
    <h4 v-else class="text-h4 mb-6">Player vs. Player</h4>
    <v-row>
      <v-col>
        <p class="text-h6 text-center mb-4">Choose deck type:</p>
        <v-radio-group v-model="deckType" class="d-flex justify-center mb-4">
          <v-radio label="People" value="people"></v-radio>
          <v-radio label="Starships" value="starships"></v-radio>
        </v-radio-group>
        <p class="text-h6 text-center mb-4">Click the button below to start the game!</p>
      </v-col>
    </v-row>
    <v-row :class="{ 'flex-column': smAndDown }">
      <v-col>
        <v-dialog v-model="isActive" max-width="800">
          <template #activator>
            <v-btn
              color="surface-variant"
              text="Start Game!"
              variant="flat"
              @click="playGame(getDeck)"
            ></v-btn>
            <v-btn
              class="ma-2"
              text="Main menu"
              color="surface-variant"
              variant="flat"
              @click="goToMainMenu"
            ></v-btn>
          </template>
          <template #default>
            <v-row>
              <v-col>
                <v-card>
                  <v-card-title class="text-center text-h4">{{ playerOne }} vs. {{ playerTwo }}</v-card-title>
                  <div v-if="gameMode === EGameMode.twoPlayers" class="text-center">
                    <div v-if="showScore">
                      <v-card-subtitle>Score:</v-card-subtitle>
                      <v-card-text>{{ playerOneScore }} : {{ playerTwoScore }}</v-card-text>
                    </div>
                    <v-skeleton-loader
                      v-else
                      type="list-item-two-line"
                      color="grey-darken-3"
                      boilerplate
                    >
                    </v-skeleton-loader>
                  </div>
                  <v-card-text v-if="showScore" class="text-center text-h4 pa-2">
                    {{ hasPlayerOneWon ? playerOne : playerTwo }} has won!
                  </v-card-text>
                  <v-skeleton-loader
                    v-else
                    type="list-item-two-line"
                    color="grey-darken-3"
                    boilerplate
                  >
                  </v-skeleton-loader>
                  <v-card-actions class="justify-center">
                    <v-btn
                      class="ma-2"
                      text="Show score"
                      color="surface-variant"
                      variant="flat"
                      :disabled="showScore"
                      @click="showScore = true"
                    ></v-btn>
                    <v-btn
                      class="ma-2"
                      text="Play Again"
                      color="surface-variant"
                      variant="flat"
                      :disabled="!showScore"
                      @click="replay"
                    ></v-btn>
                    <v-btn
                      class="ma-2"
                      text="Main menu"
                      color="surface-variant"
                      variant="flat"
                      @click="goToMainMenu"
                    ></v-btn>
                  </v-card-actions>
                </v-card>
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <v-card class="position-relative" height="550">
                  <v-img src="../public/battle.jpg" height="200" cover></v-img>
                  <div v-if="showScore">
                    <v-card-title class="mb-2">Name: {{ playerOneCard.name }}</v-card-title>
                    <v-card-text>Description: {{ playerOneCard.description }}</v-card-text>
                    <v-card-text class="mb-2">Weapon: {{ playerOneCard.weapon }}</v-card-text>
                    <v-card-text class="text-h6">
                      Resource: {{ playerOneCard.resource.type }} - {{ playerOneCard.resource.value }}
                    </v-card-text>
                  </div>
                  <v-skeleton-loader
                    v-else
                    type="paragraph, paragraph, paragraph"
                    color="grey-darken-3"
                    boilerplate
                  >
                  </v-skeleton-loader>
                </v-card>
              </v-col>
              <v-col>
                <v-card height="550">
                  <v-img src="../public/battle.jpg" height="200" cover></v-img>
                  <div v-if="showScore">
                    <v-card-title class="mb-2">Name: {{ playerTwoCard.name }}</v-card-title>
                    <v-card-text>Description: {{ playerTwoCard.description }}</v-card-text>
                    <v-card-text class="mb-2">Weapon: {{ playerTwoCard.weapon }}</v-card-text>
                    <v-card-text class="text-h6">
                      Resource: {{ playerTwoCard.resource.type }} - {{ playerTwoCard.resource.value }}
                    </v-card-text>
                  </div>
                  <v-skeleton-loader
                    v-else
                    type="paragraph, paragraph, paragraph"
                    color="grey-darken-3"
                    boilerplate
                  >
                  </v-skeleton-loader>
                </v-card>
              </v-col>
            </v-row>
          </template>
        </v-dialog>
      </v-col>
    </v-row>
  </div>
</template>
