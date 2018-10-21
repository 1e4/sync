<template>
    <div class="home">
        <div class="container-fluid my-5">
            <div class="row justify-content-center mb-5">
                <div class="col-8 text-left">
                    <h2>Welcome to {{ room }}</h2>
                    {{ this.socket_room }}
                    <h5>Room is currently playing: {{ currentVideoId }} ({{ currentVideoStatus }})</h5>
                </div>
            </div>
            <div class="row">
                <div class="col-8">
                    <div id="player-window">
                        <vue-plyr ref="player" :emit="['ready', 'play', 'pause', 'statechange']" @ready="ready"
                                  :options="{
                                    clickToPlay: false
                                  }"
                                  @play="playEvent"
                                  @pause="pauseEvent"
                                  @statechange="setVideoState"
                                    :controls="[]">
                            <div data-plyr-provider="youtube"></div>
                        </vue-plyr>
                    </div>
                    <div class="row">
                        <div class="col-6">
                            <button @click.prevent="playButton" class="btn btn-primary">Play</button>
                            <button @click.prevent="pauseButton" class="btn btn-primary">Pause</button>
                        </div>
                        <div class="col-6">
                            <form action="" class="form-inline w-100">
                                <div class="form-group">
                                    <input type="text" class="form-control" v-model="search_video_id">
                                    <button @click.prevent="switchVideo(null)" class="btn btn-primary">Switch
                                    </button>
                                </div>
                            </form>
                        </div>
                    </div>
                </div>
                <div class="col-4">
                    <div class="card">
                        <div class="card-body">
                            <div id="chat-window">
                                <ul class="list-unstyled chat-messages">
                                    <li v-for="(msg, index) in chat" v-bind:key="index" class="chat-message">
                                        <span class="author"><span v-if="msg.from_id === socket_room.owner">[o]</span>{{
                                            msg.from }}</span>
                                        <span class="message">{{ msg.message }}</span>
                                    </li>
                                </ul>
                            </div>

                            <div class="form-group">
                                <label for="message">What would you like to say?</label>
                                <input type="text" class="form-control" id="message" v-model="chat_message"
                                       @keyup.enter="sendChatMessage">
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>

    import VuePlyr from 'vue-plyr'

    export default {
        name: 'Room',
        data: function () {
            return {
                socket_room: {},
                room: this.$root.$route.params['room'],
                video_id: '',
                video_state: null,
                available_states: {
                    '-1': 'unstarted',
                    0: 'ended',
                    1: 'playing',
                    2: 'paused',
                    3: 'buffering',
                    5: 'video cued'
                },
                readyToPlay: false,
                chat: [],
                chat_message: '',
                search_video_id: 'IdlKt3SWck8',
                playerInit: false,
                socket_id: null,
                currentVideoStatus: null,
                lastVideoStatus: -1,
            }
        },
        created: function () {
            this.setup();
        },
        watch: {
            '$route': 'setup',
            playerInit: function (val) {
                console.log(val, this.video_id);
                if (val === true) {
                    this.loadVideo(this.video_id);
                }
            }
        },
        mounted() {
            this.player = this.$refs.player.player;
            // @todo fix click to play
            // this.player.options.clickToPlay = false;
        },
        computed: {
            currentVideoId() {
                return this.socket_room.currentVideo;
            }
        },
        methods: {
            setVideoState(event) {
                let code = event.detail.code;

                this.lastVideoStatus = this.currentVideoStatus;
                this.currentVideoStatus = code;

                console.log('youtube play state', code, 'last play state', this.lastVideoStatus, this.readyToPlay);

                // If it's buffering and wasn't loaded prepare to sync
                if (!this.readyToPlay) {

                    if (code === 3 && this.lastVideoStatus === -1) {
                        // Initial video load
                        console.log('waiting to buffer');
                        window.$socket.emit('waiting to buffer');
                    } else if ((code === 1 || code === 2) && this.lastVideoStatus === 3) {
                        // If playing and last one was buffering make sure everyone is synced
                        console.log('pausing as you are ready to play video, waiting for the rest');
                        this.readyToPlay = true;
                        window.$socket.emit('ready to play video');
                        this.player.pause();
                    }
                }
            },
            sendChatMessage() {
                console.log('sending message ', this.chat_message)
                this.chat.push({
                    from: 'anon',
                    message: this.chat_message,
                    from_id: this.$root.$data.socket_id
                });
                window.$socket.emit('chat message', this.chat_message)
                this.chat_message = '';
            },

            setup() {

                window.$socket.on('load room', (room) => {
                    console.log('loading room', room);
                    if (!room) this.$router.push({name: 'home'});
                    this.socket_room = room;
                    this.loadVideo(room.currentVideo)
                });

                window.$socket.on('room meta', (obj) => {
                    console.log("Updating room meta", obj)
                    this.socket_room[obj.key] = obj.value;
                });

                window.$socket.on('room closed', () => {
                    this.$root.$router.push('/')
                });

                window.$socket.on('pause video', () => {
                    if (this.readyToPlay)
                        this.player.pause();
                });

                window.$socket.on('play video', () => {
                    this.player.play();
                });

                window.$socket.on('chat message', (message) => {
                    console.log('got message', message)
                    this.chat.push(message);
                });

                window.$socket.on('switch video', (videoId) => {
                    console.log('switching video', videoId);
                    this.readyToPlay = false;
                    this.loadVideo(videoId);
                });

                window.$socket.on('play all', () => {
                    this.player.play();
                })

                window.$socket.on('buffer video', () => {
                    // Syncing/buffering video
                    this.readyToPlay = false;
                });
                window.$socket.emit('join room', this.$root.$route.params);
            },
            playEvent() {
                // window.$socket.emit('play video');
            },
            pauseEvent() {
                // window.$socket.emit('pause video');
            },
            playButton() {
                window.$socket.emit('play video');
                this.player.play();
            },
            pauseButton() {
                window.$socket.emit('pause video');
                this.player.pause();
            },
            switchVideo(id = null) {
                let videoId = id || this.search_video_id;

                this.loadVideo(videoId);

                window.$socket.emit('switch video', videoId);

                this.search_video_id = '';
            },

            ready() {
                this.playerInit = true;
            },

            loadVideo(id) {
                this.player.source = {
                    type: 'video',
                    sources: [
                        {
                            src: id,
                            provider: 'youtube',
                        },
                    ],
                };
            }
        }
    }
</script>
