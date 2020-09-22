<template>
    <div class="conversation" :style="{'height': convheight + 'px'}">
        <v-row style="height: 56px; margin-left: 30px;">
            <v-badge
                bordered
                bottom
                :color="contact.online === 'Y' ? 'green': 'grey'"
                dot
                offset-x="12"
                offset-y="16"
            >
                <v-avatar size="55">
                    <v-img :src="`storage/${contact.profile_image}`"></v-img>
                </v-avatar>
            </v-badge>
            <h5 style="margin-bottom: 0; margin-left: 10px; display: flex; align-items: center;">{{ contact ? contact.name : 'Select a Contact' }} <small>&nbsp; {{ typing }}</small></h5>
        </v-row>
        <div class="floading-div" style="position: absolute; bottom: 80px;">
            <Picker
                v-if="emoticStatus"
                set="messenger"
                @select="onInput"
                title="Pick your emoji..."
                style="width: 250px; height: 300px;"
            />
        </div>
        <div class="feed" id="chatbox" ref="feed" :style="{'height': chatheight + 'px'}">
            <ul v-if="contact">
                <li v-for="message0 in messages" :class="`message${message0.toUserId == contact.id ? ' sent' : ' received'}`" :key="message0.id">
                    <v-row v-if="message0.toUserId != contact.id">
                        <div style="width: 63px; display: flex; align-items: center;">
                            <v-avatar size="45">
                                <v-img :src="`storage/${contact.profile_image}`"></v-img>
                            </v-avatar>
                        </div>
                        <div>
                            <div class="datetime">
                                <small>{{ message0.date | dateFormat }},{{ message0.time }}</small>
                            </div>

                            <div v-if="message0.type == 'text'" class="text">
                                {{ message0.message }}
                            </div>

                            <div v-if="message0.type == 'file'" style="margin-right:1px; margin-left:1px; word-break:break-all; padding:3px 3px;">
                                <a v-if="message0.fileFormat == 'image'" :href="dataurl + message0.filePath" download :title="message0.message" target="_new">
                                    <img height="110px;" width="110px;" :src="dataurl + message0.filePath">
                                </a>

                                <div v-else>
                                    <p style="margin: 0;">{{ message0.message }}
                                        <v-btn small color="primary" icon @click="downloadfile(dataurl + message0.filePath)">
                                            <v-icon>mdi-download</v-icon>
                                        </v-btn>
                                    </p>
                                </div>
                            </div>
                        </div>
                    </v-row>

                    <v-row v-else style="display: inline-flex;">
                        <div style="margin-right: 10px">
                            <div class="datetime">
                                <small>{{ message0.date | dateFormat }},{{ message0.time }}</small>
                            </div>

                            <div v-if="message0.type == 'text'" class="text">
                                {{ message0.message }}
                            </div>

                            <div v-if="message0.type == 'file'" style="margin-right:1px; margin-left:1px; word-break:break-all; padding:3px 3px;">
                                <a v-if="message0.fileFormat == 'image'" :href="dataurl + message0.filePath" download :title="message0.message" target="_new">
                                    <img height="110px;" width="110px;" :src="dataurl + message0.filePath">
                                </a>

                                <div v-else>
                                    <p style="margin: 0;">{{ message0.message }}
                                        <v-btn small color="primary" icon @click="downloadfile(dataurl + message0.filePath)">
                                            <v-icon>mdi-download</v-icon>
                                        </v-btn>
                                    </p>
                                </div>
                            </div>
                        </div>
                        <div style="display: flex; align-items: center;">
                            <v-avatar size="45">
                                <v-img :src="`storage/${user.profile_image}`"></v-img>
                            </v-avatar>
                        </div>
                    </v-row>
                </li>
            </ul>
        </div>

        <div class="composer" style="padding: 10px;">
            <div class="message-inner">
                <div class="input-group" style="display: flex; position: relative; align-items: stretch; width: 100%">
                    <textarea
                        v-model="message"
                        class="form-control"
                        placeholder="Type message..."
                        style="border: 1px solid white; display: block; height: 44px; padding: 6px 12px; font-size: 19px; background-color: rgb(18, 18, 18); color: white;"
                        @keydown="sendMessage($event)"
                    ></textarea>
                    <span class="input-group-append">
                        <v-btn
                            @click="toggleEmotic"
                            class="mx-2"
                            fab
                            dark
                            x-small
                            color="pink"
                            style="margin-right: 0 !important; width: 44px !important; height: 44px !important;"
                        >
                            <v-icon dark>mdi-emoticon-happy</v-icon>
                        </v-btn>
                        <v-btn
                            class="mx-2"
                            fab
                            dark
                            x-small
                            color="pink"
                            style="width: 44px !important; height: 44px !important; margin-right: 0 !important;"
                            @click="$refs.fileInput.click()"
                        >
                            <v-icon>mdi-attachment</v-icon>
                            <input name="attachment1" type="file" multiple style="display:none" ref="fileInput" v-on:change="file($event)">
                        </v-btn>
                    </span>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    import { mapState } from 'vuex';
    import { Picker } from 'emoji-mart-vue';

    export default {
        data() {
            return {
                message: '',
                timeout: '',
                typing: '',
                dataurl: $('meta[name=ws_url]').attr("content"),
                convheight: $(window).height() - $('.v-toolbar__content').height() - $('.foot').height() - 20 - 80,
                chatheight: $(window).height() - $('.v-toolbar__content').height() - $('.foot').height() - 20 - 80 - 20 - 56,
                maxwidth: null,
                emoticStatus: false,
                // socket: {}
            }
        },
        computed: {
            ...mapState(["user", "contact", "messages", "socket"]),
        },
        mounted() {
            this.contact.msgCount = 0;
            this.socket.emit('getMessages', {fromUserId: this.user.id, toUserId: this.contact.id});
            this.socket.on('getMessagesResponse', this.getMessagesResponse);
            this.socket.on('addMessageResponse', this.addMessageResponse);
            this.socket.on('typing', this.typingListener);
            this.socket.on('image-uploaded', this.imageuploaded);
        },
        destroyed: function() {
            this.socket.removeListener('getMessagesResponse', this.getMessagesResponse);
            this.socket.removeListener('addMessageResponse', this.addMessageResponse);
            this.socket.removeListener('typing', this.typingListener);
        },
        methods: {
            sendMessage: function(e) {
                if(e.keyCode === 13){
                    e.preventDefault();
                    if (this.message == '') {
                        return;
                    }

                    // Update contact info
                    this.contact.message = this.message;
                    let timediff = "secs ago";
                    this.contact.date = timediff;

                    // Send the message
                    let messagePacket = this.createMsgObj('text', '', this.message);
                    this.socket.emit('addMessage', messagePacket);
                    this.messages.push(messagePacket);
                    this.message = '';
                    this.scrollToBottom();
                } else {
                    if((e.keyCode != 116) && (e.keyCode != 82 && !e.ctrlKey)){
                        this.socket.emit('typing', {typing:'typing...', socket_id:this.contact.socket_id});
                        clearTimeout(this.timeout);
                        this.timeout = setTimeout(this.timeoutFunction, 500);
                    }
                }
            },
            timeoutFunction: function(){
                this.socket.emit("typing", {typing:false, socket_id:this.contact.socket_id});
            },
            createMsgObj : function(type, fileFormat, message){
                return {
                    type: type,
                    fileFormat: fileFormat,
                    filePath: '',
                    fromUserId: this.user.id,
                    toUserId: this.contact.id,
                    toSocketId: this.contact.socket_id,
                    message: message,
                    time: new moment().format("hh:mm A"),
                    date: new moment().format("YYYY-MM-DD")
                }
            },
            addMessageResponse: function(data){
                let addMsg = data.addMsg;
                if (data && data.fromUserId == this.contact.id) {
                    this.messages.push(data);
                    this.scrollToBottom();
                }
            },
            typingListener: function(data){
                if (data.typing && data.to_socket_id == this.contact.socket_id) {
                    this.typing = " is "+data.typing;
                } else {
                    this.typing = "";
                }
            },
            getMessagesResponse(data) {
                if (data.toUserId == this.contact.id) {
                    this.$store.dispatch("messagesfcn", data.result);
                    this.$nextTick(function () {
                        this.scrollToBottom();
                    });
                }
            },
            imageuploaded: function(data) {
                if (data && data.toUserId == this.contact.id) {
                    $(".overlay").parent().parent().remove();
                    this.messages.push(data);
                    this.scrollToBottom();
                }
            },
            file: function(event) {
                var file = event.target.files[0];
                if (this.validateSize(file)) {
                    let fileFormat = file.type.split('/')[0];
                    let reader  = new FileReader();
                    reader.onload = function () {
                        let messagePacket = this.createMsgObj('file', fileFormat, reader.result);
                        messagePacket['fileName'] = file.name;
                        this.socket.emit('upload-image', messagePacket);
                    }.bind(this);
                    reader.readAsArrayBuffer(file);
                } else {
                    event.target.value = "";
                    alert('File size exceeds 10 MB');
                }
            },
            validateSize: function(file) {
                var fileSize = file.size / 1024 / 1024; // in MB
                if (fileSize > 10) {
                    return false;
                }
                return true;
            },
            scrollToBottom() {
                // $('#chatbox').stop().animate({ scrollTop: $('#chatbox')[0].scrollHeight}, 1);
                setTimeout(() => {
                    this.$refs.feed.scrollTop = this.$refs.feed.scrollHeight - this.$refs.feed.clientHeight;
                }, 50);
            },
            onInput(e) {
                if (!e) {
                    return false;
                };
                this.message = this.message + e.native;
                let messagePacket = this.createMsgObj('text', '', this.message);
            },
            toggleEmotic() {
                this.emoticStatus = !this.emoticStatus
            },
            downloadfile(filelink) {
                axios({
                    url: filelink,
                    method: 'GET',
                    responseType: 'blob'
                })
                .then((response) => {
                     var fileURL = window.URL.createObjectURL(new Blob([response.data]));
                     var fileLink = document.createElement('a');

                     fileLink.href = fileURL;
                     fileLink.setAttribute('download', 'file.pdf');
                     document.body.appendChild(fileLink);

                     fileLink.click();
                })
                .catch(() => console.log('error occured'));
            }
        },
        filters: {
            dateFormat: function(value) {
                return new moment(value).format("DD-MM-YYYY")
            }
        },
        components: {
            Picker
        }
    }
</script>

<style lang="scss" scoped>
    .conversation {
        padding: 20px 15px 0 15px;

        .row {
            margin: 0;
        }

        .floading-div {
            position: fixed;
        }

        .feed {
            height: 100%;
            overflow: auto;
            margin: 5px;

            ul {
                list-style-type: none;
                padding: 5px;

                li {
                    &.message {
                        padding: 12px 0;
                        width: 100%;

                        .text {
                            max-width: 400px;
                            border-radius: 5px;
                            padding: 12px;
                            display: inline-block;
                        }

                        &.received {
                            text-align: left;

                            .text {
                                background: #1da058;
                            }
                        }

                        &.sent {
                            text-align: right;

                            .text {
                                background: #4b30c4;
                            }

                        }
                    }
                }
            }
        };
        .composer {
            .row {
                    .v-input__append-outer {
                        margin: 0px !important;
                    }
            }
        };
        .v-badge--dot .v-badge__badge {
            border-radius: 5px;
            height: 10px;
            min-width: 0;
            padding: 0;
            width: 10px;
        }
    }
</style>
