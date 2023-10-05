<script>
import axios from 'axios'

import dayjs from 'dayjs';

export default {
    data() {
        return {
            news: null,
            userProfile: null,
            user: null,
            localStorage: localStorage,
            readyToAdd: true,
            content: ""
        };
    },
    methods: {
        async loadUserProfile() {
            let response = await axios.get(`/find/user`, { params: { id: this.$route.params.id } });
            this.userProfile = response.data;
        },
        async loadUser() {
            let response = await axios.get(`/find/me`);
            this.user = response.data;
        },
        getAge() {
            let day = dayjs();
            let fullBirthday = dayjs(this.userProfile.birthday);
            let birthday = fullBirthday.format('YYYY');
            return day.format('YYYY') - birthday;
        },
        getAgeWord(number) {
            let declension = ['год', 'года', 'лет'];
            let cases = [2, 0, 1, 1, 1, 2];
            return declension[((number % 100 > 4 && number % 100 < 20) ? 2 : cases[(number % 10 < 5) ? number % 10 : 5])];
        },
        async addFriend() {
            if (this.readyToAdd) {
                this.readyToAdd = false;
                let response = await axios.post(`/addFriend`, {
                    userId: this.user._id,
                    userFriendId: this.userProfile._id
                });
                if (response.status == 200) {
                    await this.loadUserProfile();
                    await this.loadUser();
                }
                this.readyToAdd = true;
            }
        },
        async loadNews() {
            let response = await axios.get(`/news/my`, {
                params: {
                    id: this.userProfile._id
                }
            })
            this.news = response.data
        },
        goDialogs() {
            this.$router.push("/dialogs");
        },
        goComments(item) {
            this.$router.replace(`/news-` + item._id)
        },
        getDate(date) {
            let day = dayjs(date)
            return day.format(`HH:mm DD.MM.YY`)
        },
        goProfile(user) {
            this.$router.replace(`/profile-` + user._id)
        },
        async deleteNews(id) {
            if (this.userProfile._id === this.user._id) {
                let response = await axios.delete("/news", {data: { id: id }})
                if (response.status == 200) {
                    this.loadNews()
                }
            }
        },
        async addNews() {
            if (this.userProfile._id === this.user._id) {
                let response = await axios.post("/news", { content: this.content })
                if (response.status == 200) {
                    await this.loadNews()
                    this.content = ""
                }
            }
        }
    },
    async created() {
        await this.loadUserProfile();
        await this.loadUser();
        await this.loadNews()
    },
    watch: {
        $route() {
            if (this.$route.name == "profile") {
                this.loadUserProfile();
            }
        }
    }
}
</script>

<template>
    <section class="info__body" v-if="user && userProfile && news">
        <div class="info__page-cover"></div>
        <div class="info__data">
            <div class="info__avatar" :style="{
                backgroundImage: 'url(' + userProfile.avatar + ')',
                backgroundPosition: 'center center',
                backgroundSize: 'cover'
            }" :class="{ 'empty-avatar': !userProfile.avatar }"></div>
            <div class="info__username">
                <h2 v-html="userProfile.username"></h2><img v-if="userProfile.gender"
                    :src="(userProfile.gender == 'male') ? 'https://i.ibb.co/SPbnhrg/male.png' : 'https://i.ibb.co/dfd5JYp/female.png'">
            </div>
            <button v-if="!(userProfile._id == user._id) && !(user.friends.includes(userProfile._id))" @click="addFriend()">
                <img v-if="readyToAdd" src="@/assets/addfriend.svg" alt="add friend">
                <img v-else src="@/assets/loading.gif" alt="add friend">
            </button>
            <button v-if="!(userProfile._id == user._id) && (user.friends.includes(userProfile._id))" @click="goDialogs()">
                <img src="@/assets/typing.png" alt="go dialog">
            </button>
        </div>

        <div class="info__age">
            <p>Возраст:
                <span v-if="userProfile.birthday">{{ getAge() }} {{ getAgeWord(getAge()) }}</span>
                <span v-else>Не указан</span>
            </p>
        </div>

        <div class="info__about-me">
            <h5>О себе</h5>
            <p v-html="userProfile.aboutme"></p>
        </div>

        <div class="news">
            <form class="news__form" @submit.prevent="addNews()">
                <textarea type="text" placeholder="Что нового..." v-model="content"></textarea>
                <button type="submit">Опубликовать</button>
            </form>
        </div>

        <div class="news__list">
            <article v-for="(item, index) in news" class="new__body" :key="item._id">
                <div class="new__header" @click="goProfile(item.author)">
                    <div class="new__avatar" :style="{
                        backgroundImage: 'url(' + item.author.avatar + ')',
                        backgroundPosition: 'center center',
                        backgroundSize: 'cover'
                    }"></div>

                    <p class="new__author" v-html="item.author.username"></p>

                    <img v-if="userProfile._id === user._id" src="@/assets/delete.png" alt="delete icon"
                        @click="deleteNews(item._id)">
                </div>

                <div class="new__main" v-html="item.content"></div>

                <div class="new__footer">
                    <div class="new__reactions">
                        <div class="new__likes" @click="likeNews(item)">
                            <img
                                :src="(user.likedNews.includes(item._id) ? `https://i.ibb.co/8zct618/liked.png` : `https://i.ibb.co/vd3TQ5p/unliked.png`)" />
                            <span class="new__likes-number">{{ item.likes }}</span>
                        </div>

                        <div class="new__comments" @click="goComments(item)">
                            <img src="@/assets/comment.png" alt="comment button" />
                            <span class="new__comments-number">{{ item.comments.length }}</span>
                        </div>
                    </div>

                    <div class="new__createdAt">{{ getDate(item.createdAt) }}</div>
                </div>
            </article>
        </div>
    </section>
</template>

<style lang="scss" scoped>
%end {
    border-radius: 0 0 1rem 1rem;
    border-bottom: 1px solid #0005;

    position: relative;
    bottom: 48px;
}

main {
    min-width: 100%;
    min-height: 100%;

    display: block;

    .info__body {
        .info__page-cover {
            width: 100%;
            height: 100px;

            background: linear-gradient(1.1turn, #c7b1fd, #f6b8ff, #e7ffd5, #affffc, #99ffcc);
            background-size: 300% 300%;
            animation: back 14s infinite alternate;

            @keyframes back {
                25% {
                    background-position: 0 50%;
                }

                50% {
                    background-position: 100% 50%;
                }

                100% {
                    background-position: 0 50%;
                }
            }

            z-index: 0;
        }

        .info__data {
            display: flex;
            align-items: end;

            @extend %end;

            .info__avatar {
                padding: 3px;
                margin: 10px;

                height: 80px;
                width: 80px;

                overflow: hidden;

                border-radius: 50%;

                background-color: #fff;

                border: 4px solid #fff;
            }

            .empty-avatar {
                background-image: url("@/assets/empty account.svg") !important;
            }

            .info__username {
                display: flex;
                align-items: center;
                gap: 5px;

                img {
                    height: 2rem;
                }
            }

            button {
                background-color: #fff;

                margin: 0.83em 0.83em 0.83em auto;
                font-size: 1.5em;

                border: 1px dashed #0006;
                border-radius: 0.375rem;

                cursor: pointer;

                &:hover {
                    border-style: solid;
                }

                img {
                    height: 20px;
                }
            }
        }

        .info__age {
            padding: 14px 18px;

            @extend %end;

            p {
                padding: 0;
                margin: 0 0 5px 0;

                font-size: 16px;
                font-weight: 550;

                color: #0007;

                span {
                    padding: 4px 0;
                    margin: 0;

                    font-size: 18px;

                    color: #000;
                }
            }
        }

        .info__about-me {
            padding: 14px 18px;

            @extend %end;

            h5 {
                padding: 0;
                margin: 0;

                font-size: 16px;
                font-weight: 550;

                color: #0007;
            }

            p {
                padding: 5px 0;
                margin: 0;

                font-size: 18px;

                word-wrap: break-word;
            }
        }

        .news {
            @extend %end;

            margin: 10px 0;
            padding-bottom: 20px;

            .news__form {
                display: flex;
                flex-direction: column;
                gap: 10px;

                textarea {
                    resize: vertical;

                    height: 70px;

                    border: 0px;

                    font-size: 16px;

                    margin: 0 20px;
                    padding: 10px;

                }

                button {
                    margin: 0 20px;
                    padding: 5px;

                    font-size: 18px;

                    background-color: pink;

                    border-radius: .5rem;
                    border: 0px;

                    cursor: pointer;
                }
            }
        }
    }
}

.news__list {
    display: flex;
    flex-direction: column;
    align-items: start;
    gap: 26px;

    @extend %end;

    .new__body {
        display: flex;
        flex-direction: column;
        justify-content: start;

        border: 1px solid #0006;

        width: 100%;

        &:last-child {
            border-bottom: 0px;
        }


        .new__header {
            display: flex;
            align-items: center;
            gap: 14px;

            border-bottom: 1px solid #0004;

            margin: 4px 8px;
            padding: 0 8px;

            .new__avatar {
                height: 50px;
                width: 50px;

                border-radius: 50%;
            }

            p {
                font-size: 20px;
            }

            img {
                margin-left: auto;
                cursor: pointer;
            }
        }

        .new__main {
            margin: 10px 20px;

            img {
                max-width: 100%;
            }
        }

        .new__footer {
            display: flex;
            justify-content: space-between;
            align-items: center;

            padding: 6px 10px;
            margin: 0 14px;

            border-top: 1px solid #0004;

            .new__reactions {
                display: flex;
                justify-content: start;
                align-items: center;
                gap: 20px;

                .new__likes {
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    gap: 6px;

                    cursor: pointer;

                    img {
                        height: 30px;

                        transition: opacity .2s;

                        background-color: #fff;
                    }
                }

                .new__comments {
                    display: flex;
                    justify-content: center;
                    align-items: center;
                    gap: 6px;

                    cursor: pointer;

                    img {
                        height: 28px;
                    }
                }
            }
        }
    }
}
</style>