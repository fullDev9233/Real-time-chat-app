<template>
    <div>
        <!-- <div class="button-wrapper">
			<span class="button" @click="$refs.file.click()">
				<input type="file" ref="file" @change="uploadImage($event)" accept="image/*">
				Upload image
			</span>
		</div>

        <v-avatar size="120" v-if="!loaded">
            <img style="background:white; border:5px solid white;" :src="avatar">
        </v-avatar> -->

        <div class="avatar-upload" v-if="!loaded">
            <div class="avatar-edit">
                <input type='file' id="imageUpload" @change="uploadImage($event)" accept=".png, .jpg, .jpeg" />
                <label for="imageUpload"></label>
            </div>
            <div class="avatar-preview">
                <div id="imagePreview" :style="{ 'background-image': 'url('+ avatar +')' }"></div>
            </div>
        </div>

        <div v-if="loaded" style="display: flex; justify-content: center;">
            <Cropper
                classname="upload-example-cropper"
                :src="avatar"
                ref="cropper"
            />
        </div>

        <a href="#" class="btn btn-success" @click.prevent="saveImage" v-if="loaded">Save</a>
        <a href="#" class="btn btn-danger" @click.prevent="cancel" v-if="loaded">Cancel</a>
    </div>
</template>

<script>
    import { mapState } from 'vuex';
    import { Cropper } from 'vue-advanced-cropper';

    export default {
        props: ['user'],
        data() {
            return {
                avatar: `storage/${this.user.profile_image}`,
                loaded: false,
                file: null,
                filename: null
            }
        },
        methods: {
            uploadImage(event) {
                let input = event.target;
                if (input.files && input.files[0]) {
                    let imgfile = input.files[0];
                    this.filename = imgfile.name;
                    this.read(imgfile);
                }
            },
            crop() {
                const {coordinates, canvas} = this.$refs.cropper.getResult();
                this.coordinates = coordinates;
                let image = canvas.toDataURL();
                let form = new FormData();
                form.append('image', image);
                form.append('imagename', this.filename)
                this.file = form;
            },
            saveImage() {
                this.crop();
                axios.post('/api/user', this.file, {
                        headers: {
                            'Authorization': `Bearer ${this.user.token}`,
                            'Content-Type': 'multipart/form-data'
                        }
                    })
                    .then(res =>
                        {
                            this.$toasted.show('Avatar is successfully uploaded.', {
                                action : {
                                    text : 'Close',
                                    onClick : (e, toastObject) => {
                                        toastObject.goAway(0);
                                    }
                                },
                                type: 'success'
                            }),
                            this.user.profile_image = this.filename;
                            sessionStorage.setItem("user", JSON.stringify(this.user));
                            this.loaded = false
                        }
                    )
                    .catch(function (error) {
                        console.log(error);
                    });
            },
            read(image) {
                let reader = new FileReader();
                reader.readAsDataURL(image);
                reader.onload = e => {
                    this.avatar = e.target.result;
                };
                this.loaded = true;
            },
            cancel() {
                this.avatar = `storage/${this.user.profile_image}`;
                this.loaded = false;
            }
        },
        components: {
            Cropper
        }
    }
</script>

<style lang="scss">
    .avatar-upload {
        position: relative;
        max-width: 205px;
        margin: 8px auto;
        .avatar-edit {
            position: absolute;
            right: 22px;
            z-index: 1;
            top: 10px;
            input {
                display: none;
                + label {
                    display: inline-block;
                    width: 34px;
                    height: 34px;
                    margin-bottom: 0;
                    border-radius: 100%;
                    background: #5efa03;
                    border: 1px solid transparent;
                    box-shadow: 0px 2px 4px 0px rgba(0, 0, 0, 0.12);
                    cursor: pointer;
                    font-weight: normal;
                    transition: all .2s ease-in-out;
                    &:hover {
                        background: #5efa03;
                        border-color: #d6d6d6;
                    }
                    &:after {
                        content: "\f040";
                        font-family: 'FontAwesome';
                        color: #757575;
                        position: absolute;
                        top: 6px;
                        left: 0;
                        right: 0;
                        text-align: center;
                        margin: auto;
                    }
                }
            }
        }
        .avatar-preview {
            width: 192px;
            height: 192px;
            position: relative;
            border-radius: 100%;
            border: 6px solid #F8F8F8;
            box-shadow: 0px 2px 4px 0px rgba(0, 0, 0, 0.1);
            > div {
                width: 100%;
                height: 100%;
                border-radius: 100%;
                background-size: cover;
                background-repeat: no-repeat;
                background-position: center;
            }
        }
    }

    .upload-example-cropper {
        border: solid 1px #EEE;
        height: 200px;
        width: 200px;
    }

    .button-wrapper {
        display: flex;
        justify-content: center;
        padding-bottom: 10px;
    }

    .button {
        color: white;
        font-size: 14px;
        padding: 5px 10px;
        background: #3fb37f;
        cursor: pointer;
        transition: background 0.5s;
    }

    .button:hover {
        background: #38d890;
    }

    .button input {
        display: none;
    }

    /* .vue-rectangle-stencil .vue-rectangle-stencil--movable {
        width: 150px !important;
        height: 150px !important;
        left: 15px !important;
        top: 15px !important;
    } */

    .cropper {
        height: 200px;
        width: 200px
    }
</style>
