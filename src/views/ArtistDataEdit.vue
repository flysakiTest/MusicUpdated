<template>
	<div class="view_edit">
		<form>
			<div id="edit_artist">
				<label>
					<span>アーティスト</span>
					<input v-model="artistName" type="text"/>
				</label>
				<label>
					<span>ID</span>
					<input v-model="artistId" type="text" style="width: 5em;"/>
				</label>
				<div id="edit_control_btn_ctn">
					<span @click="is_show_artist_detail = !is_show_artist_detail">！</span>
					<span @click="showAllDetailSwitch(false)">▲</span>
					<span @click="showAllDetailSwitch(true)">▼</span>
					<span @click="musicWorkAdd()">＋</span>
				</div>
			</div>
			<div id="edit_artist_info" v-show="is_show_artist_detail">
				<label>
					<span>最新作登録日</span>
					<input v-model="dateUpdated" type="text" class="input_date"/>
				</label>
				<label>
					<span>最新チェックした日</span>
					<input v-model="dateChecked" type="text" class="input_date"/>
				</label>
				<div class="inputWithTitle" style="width: 100%; height: fit-content;">
					<span class="leftBorderLabel">アーティスト リンク集と備考</span>
					<textarea class="url_note" v-model="artistNoteUrls" />
				</div>
			</div>
			<div id="edit_work_list">
				<MusicWorkEditor
						v-for="work in musicWorksDesc"
						:key="artistId + '_w' + work.key_in_page"
						:work-json="work"
						ref="musicWorks"

						@delete="musicWorkRemove(work.key_in_page)"
				/>
			</div>
		</form>
		<div id="io">
			<div>
				<div>JSON作成</div>
				<label>
					<input type="checkbox" v-model="is_user_added"/>
					<span>今回は最新作を<b>登録</b>しました</span>
				</label>
				<label>
					<input type="checkbox" v-model="is_user_checked"/>
					<span>新作情報チェックしました</span>
				</label>
				<button @click="getJson">作成</button>
			</div>
			<div>
				<textarea cols="100" rows="8" ref="ioTextarea"></textarea>
			</div>
		</div>


	</div>
</template>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
	#io{
		margin-top: 3em;
	}

	.view_edit {
		max-width: 60em;
		margin: 1em auto;
	}

	.view_edit > form > * {
		margin-bottom: 1em;
	}

	#edit_control_btn_ctn {
		display: inline-block;
		float: right;
		user-select: none;
	}

	#edit_control_btn_ctn > span {
		padding: 0.1em 0.2em;
		margin: 0 0.2em;
		line-height: 1.1em;
		width: 1.5em;
		border: 1px solid;
	}

	#edit_control_btn_ctn > span:hover {
		color: #42b983;
	}

	#edit_artist *, #edit_artist_info * {
		margin-right: 0.5em;
	}

	#edit_artist_info > * {
		margin-top: 0.6em;
	}

	#edit_artist_info {
		border: 1px solid #42b983;
	}

	.url_note {
		width: calc(100% - 1.6em);
		font-size: inherit;
		height: 6em;
	}

	label.workTitle {
		width: 60%;
	}

	label.workTitle > * {
		width: 100%;
	}

	input.input_date {
		width: 6.5em;
	}

	.tags *:hover {
		color: #42b983;
	}

</style>

<script>
	import MusicWorkEditor from "../components/MusicWorkEditor";
	import axios from 'axios'

	export default {
		beforeRouteEnter (route, redirect, next) {
			if(route.params.oid){
				let d = new Date();
				let artistId = route.params.oid;
				axios.get(process.env.VUE_APP_URL_DATA + artistId + '.json?time=' + d).then(function (res) {
					route.params.jsonRaw = JSON.stringify(res.data);
					next();
				}).catch(function () {
					route.params.jsonRaw = '{}';
					next();
				})
			}
			else if(!route.params.jsonRaw){
				alert('no input content');
				next('/import');
			} else {
				next();
			}
		},
		components: {MusicWorkEditor},
		name: 'ArtistDataEditor',
		props: {
			msg: String
		},
		data() {
			return {
				artistId: this.$route.params.oid,

				artistName: '',
				musicWorks: [],
				dateUpdated: '',
				dateChecked: '',
				artistNoteUrls: '',

				output_json: '',
				is_show_artist_detail: false,
				is_user_added: false,
				is_user_checked: false,
				key_in_page_current: 0
			}
		},
		computed: {
			musicWorksDesc: function () {
				return this.musicWorks.slice().reverse();
			}
		},
		created() {
			this.importJson(this.$route.params.jsonRaw);
		},
		methods: {
			importJson: function (jsonRaw) {
				let inputArtistData = JSON.parse(jsonRaw);

				if (
					!inputArtistData.name
					|| !inputArtistData.name.length
				) {
					alert('nameの書式を確認してください');
					return;
				}
				if (
					!inputArtistData.musicWorks
					|| !inputArtistData.musicWorks.length
				) {
					alert('musicWorksの書式を確認してください');
					return;
				}
				if (
					!inputArtistData.dateUpdated
					|| !inputArtistData.dateUpdated.match(/\d{4}-\d{2}-\d{2}/)
				) {
					alert('dateUpdatedの書式を確認してください');
					return;
				}
				if (
					!inputArtistData.dateChecked
					|| !inputArtistData.dateChecked.match(/\d{4}-\d{2}-\d{2}/)
				) {
					alert('dateCheckedの書式を確認してください');
					return;
				}

				this.artistName = inputArtistData.name;
				this.artistNoteUrls = inputArtistData.note_urls;
				this.musicWorks = [];
				this.dateUpdated = inputArtistData.dateUpdated;
				this.dateChecked = inputArtistData.dateChecked;

				inputArtistData.musicWorks.sort(this.musicWorkSort);
				inputArtistData.musicWorks.reverse();
				for (let i = 0; i < inputArtistData.musicWorks.length; i++) {
					let work = inputArtistData.musicWorks[i];
					this.musicWorkAdd({
						title: work.title,
						artist_name: work.artist_name === this.artistName ? '' : work.artist_name,
						release_date: work.release_date,
						url_note: work.url_note,
						tags: work.tags,
					});
				}
			},
			getJson: function () {
				let outputArtistData = {
					name: '',
					note_urls: '',
					musicWorks: [],
					dateUpdated: '',
					dateChecked: '',
				}

				//name
				outputArtistData.name = this.artistName;

				//note_urls
				outputArtistData.note_urls = this.artistNoteUrls;

				//musicWorks
				this.$refs.musicWorks.sort(this.musicWorkSort);
				for (let i = 0; i < this.$refs.musicWorks.length; i++) {
					let work = this.$refs.musicWorks[i];
					outputArtistData.musicWorks.push(work.workOutput);
				}

				let today = new Date();
				today =
					today.getFullYear()
					+ "-" + (today.getMonth() + 1).toString().padStart(2, '0')
					+ "-" + today.getDay().toString().padStart(2, '0');
				//dateUpdated
				outputArtistData.dateUpdated = this.is_user_added ? today : this.dateUpdated;

				//dateChecked
				outputArtistData.dateChecked = this.is_user_checked ? today : this.dateChecked;

				this.$refs.ioTextarea.value = JSON.stringify(outputArtistData, null, "\t");
			},
			musicWorkSort: function (a, b) {
				if (a.release_date === b.release_date) {
					return 0;
				} else if (a.release_date > b.release_date) {
					return -1;
				} else {
					return 1;
				}
			},
			showAllDetailSwitch: function (flag) {
				this.$refs.musicWorks.forEach(function (work) {
					work.isDetailShowed = flag;
				})
			},
			musicWorkAdd: function (workInput) {
				let work = workInput ? workInput : {
					title: '',
					artist_name: '',
					release_date: '',
					url_note: '',
					tags: [],
				};
				work.key_in_page = this.key_in_page_current++;
				this.musicWorks.push(work);
			},
			musicWorkRemove: function (key) {
				let index = this.musicWorks.findIndex(work => work.key_in_page === key);
				this.musicWorks.splice(index, 1);
			}
		}
	}
</script>
