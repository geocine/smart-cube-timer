<template>
	<v-container fill-height grid-list-lg pa-0>
		<v-layout column fill-height ma-0>
			<v-flex class="controls">
				<v-alert
					v-if="isFirstSolve"
					v-model="isDescriptionShown"
					:type="phase === 'prescramble' ? 'warning' : 'info'"
				>
					<span v-if="phase === 'connect'">
						Make sure GiiKER is solved state, and press "Connect Cube" to link cube.
					</span>
					<span v-if="phase === 'prescramble'">
						Solve the cube before starting scramble.
					</span>
					<span v-if="phase === 'scramble'">
						Follow the scramble.
					</span>
					<span v-if="phase === 'inspect'">
						Now start solving when you're ready.
					</span>
				</v-alert>
				<v-btn
					v-if="!isGiikerConnected"
					:disabled="isConnecting"
					:loading="isConnecting"
					color="success"
					large
					@click="onClickConnect"
				>
					Connect Cube
				</v-btn>
				<div
					v-if="phase === 'scramble'"
					class="scramble"
				>
					<span
						v-for="(move, index) in scrambleMoves"
						:key="index"
						:style="{color: move.grey ? '#CCC' : ''}">
						{{move.text}}
					</span>
				</div>
				<div class="timer">
					<v-btn
						v-if="phase === 'scramble' && previousSolve !== null"
						icon
						flat
						disabled
						class="ma-0"
					/>
					{{displayTime}}
					<v-dialog
						v-if="phase === 'scramble' && previousSolve !== null"
						v-model="isDeleteDialogOpen"
					>
						<v-btn
							slot="activator"
							icon
							flat
							color="red lighten-2"
							class="ma-0"
						>
							<v-icon dark>delete</v-icon>
						</v-btn>
						<v-card>
							<v-card-text>
								Delete solve?
							</v-card-text>
							<v-divider/>
							<v-card-actions>
								<v-spacer/>
								<v-btn
									color="primary"
									flat
									@click="isDeleteDialogOpen = false"
								>
									Cancel
								</v-btn>
								<v-btn
									color="red lighten-1"
									flat
									@click="onClickDelete"
								>
									Delete
								</v-btn>
							</v-card-actions>
						</v-card>
					</v-dialog>
				</div>
				<div
					v-if="phase === 'solve'"
					class="timer-controls"
				>
					<v-btn
						flat
						small
						color="error"
						class="ma-0"
						@click="onClickReset"
					>
						Reset
					</v-btn>
				</div>
				<div
					v-if="phase === 'scramble' && !isFirstSolve"
					class="solve-infos"
				>
					<span class="solve-info subheading">
						{{moveCount}} turns
					</span>
					<span class="solve-info subheading">
						{{speed}} tps
					</span>
				</div>
			</v-flex>
			<v-flex id="stages" class="times">
				<stages
					:stages="stages"
					:mode="mode"
					:time="time"
					:cross="cross"
					:is-xcross="isXcross"
					:oll="oll"
					:is-oll2look="isOll2Look"
					:pll="pll"
					:pll-looks="pllLooks"
					:cll="cll"
				/>
			</v-flex>
		</v-layout>
		<v-dialog v-model="isDialogOpen" max-width="400">
			<v-card>
				<v-card-title class="headline">Your browser is not supported</v-card-title>
				<v-card-text>
					It seems your browser doesn't support Web Bluetooth API.
					So this timer cannot communicate with GiiKER and totally not works.
				</v-card-text>
				<v-card-text v-if="platform.startsWith('Win')">
					I guess you are using <strong>Windows</strong>.
					You can try Chrome Dev with new Bluetooth implementation but it's very beta.
					If you are so smart to try out beta, follow <a
						target="_blank"
						href="https://github.com/hakatashi/smart-cube-timer/wiki/Windows-Guide">this instruction</a> at your own risk.
				</v-card-text>
				<v-card-text v-if="platform.startsWith('iP')">
					I guess you are using <strong>{{platform}}</strong>.
					Some people says that this timer works with <a
						target="_blank"
						href="https://itunes.apple.com/us/app/webble/id1193531073">WebBLE browser</a> ($1.99) but I don't guarantee.
						Try at your own risk.
				</v-card-text>
				<v-card-text v-if="platform.startsWith('Mac')">
					I guess you are using <strong>Mac</strong>.
					Download <a
						target="_blank"
						href="https://www.google.com/chrome/">latest Google Chrome</a> and it should help.
				</v-card-text>
				<v-card-text v-if="platform.startsWith('Android') || platform.match(/linux/i)">
					I guess you are using <strong>Android</strong>.
					Download <a
						target="_blank"
						href="https://play.google.com/store/apps/details?id=com.android.chrome">latest Google Chrome</a> and it should help.
				</v-card-text>
				<v-card-actions>
					<v-spacer/>
					<v-btn
						color="green darken-1"
						flat="flat"
						@click="isDialogOpen = false"
					>
						Close
					</v-btn>
				</v-card-actions>
			</v-card>
		</v-dialog>
		<v-snackbar
			v-model="isSnackbarShown"
			:timeout="5000"
			color="error"
			bottom
			multi-line
		>
			{{snackbar}}
			<v-btn
				flat
				@click="isSnackbarShown = false"
			>
				Close
			</v-btn>
		</v-snackbar>
	</v-container>
</template>

<script>
import {deleteSolve, saveSolve} from '~/lib/db.js';
import {
	findCross,
	findRouxBlock,
	formatTime,
	getInspectionTime,
	getNextStage,
	getRotation,
	isStageSatisfied,
} from '~/lib/utils.js';
import GiiKER from '~/lib/giiker.js';
import MoveSequence from '~/lib/MoveSequence.js';
import NoSleep from 'nosleep.js';
import Stages from '~/components/Stages.vue';
import assert from 'assert';
import config from '~/lib/config.js';
import sample from 'lodash/sample';
import scrambles from '~/lib/scrambles.json';
import scrambo from 'scrambo/lib/scramblers/333';
import sumBy from 'lodash/sumBy';
import uniq from 'lodash/uniq';

export default {
	components: {
		Stages,
	},
	data() {
		return {
			platform: '',
			mode: 'cfop',
			cross: null,
			rouxBlock: null,
			isGiikerConnected: null,
			startTime: null,
			time: 0,
			phase: 'connect',
			isConnecting: false,
			isDescriptionShown: true,
			placeholderMoves: [],
			scramble: null,
			solveSequence: null,
			cubeStage: null,
			stages: {},
			oll: null,
			previousSolve: null,
			isOll2Look: false,
			pll: null,
			pllLooks: [],
			cll: null,
			snackbar: '',
			isSnackbarShown: false,
			isFirstSolve: true,
			isDialogOpen: false,
			isDeleteDialogOpen: false,
		};
	},
	computed: {
		scrambleMoves() {
			if (this.scramble === null) {
				return [];
			}

			assert(this.placeholderMoves.length >= this.scramble.moves.length);

			const greyedMovesCount = this.placeholderMoves.length - this.scramble.moves.length;

			return this.placeholderMoves.map((move, index) => {
				if (index < greyedMovesCount) {
					return {
						text: MoveSequence.moveToString(move),
						grey: true,
					};
				}

				const actualMove = this.scramble.moves[index - greyedMovesCount];

				if (index === greyedMovesCount) {
					if (move.face === actualMove.face) {
						return {
							text: MoveSequence.moveToString(move),
							grey: false,
						};
					}
				}

				return {
					text: MoveSequence.moveToString(actualMove),
					grey: false,
				};
			});
		},
		isXcross() {
			return this.stages.f2l1 && this.stages.f2l1.time !== null && this.stages.f2l1.sequence.length === 0;
		},
		displayTime() {
			return formatTime(this.time);
		},
		moveCount() {
			return sumBy(Object.values(this.stages), ({time = null, sequence}) => time === null ? 0 : sequence.length);
		},
		speed() {
			if (this.time === 0) {
				return (0).toFixed(2);
			}

			return (this.moveCount / (this.time / 1000)).toFixed(2);
		},
	},
	created() {
		if (process.browser) {
			this.noSleep = new NoSleep();
		}
		this.isGiikerConnected = GiiKER.isConnected;
		if (GiiKER.isConnected) {
			if (GiiKER.cube.isSolved()) {
				this.phase = 'scramble';
			} else {
				this.phase = 'prescramble';
			}
			GiiKER.on('move', this.onGiikerMove);
		}
	},
	mounted() {
		const scramble = sample(scrambles.sheets[0].scrambles);
		this.scramble = MoveSequence.fromScramble(scramble, {mode: 'reduction'});
		this.initialScramble = MoveSequence.fromScramble(scramble, {mode: 'reduction'});
		this.turns = new MoveSequence([], {mode: 'raw'});
		this.placeholderMoves = this.scramble.moves.map((move) => ({...move}));

		this.isDialogOpen = !navigator.bluetooth && typeof BluetoothDevice === 'undefined';
		this.platform = navigator.platform;
	},
	destroyed() {
		if (this.interval) {
			clearInterval(this.interval);
		}
		if (GiiKER.isConnected) {
			GiiKER.off('move', this.onGiikerMove);
		}
		if (this.noSleep) {
			this.noSleep.disable();
		}
	},
	methods: {
		onClickConnect() {
			if (this.isConnecting) {
				return;
			}

			if (this.noSleep) {
				this.noSleep.enable();
			}

			this.isConnecting = true;

			GiiKER.connect().then(() => {
				this.isGiikerConnected = true;
				GiiKER.on('move', this.onGiikerMove);
				this.phase = 'scramble';
			}, (error) => {
				this.isSnackbarShown = true;
				this.snackbar = error.message;
				this.isConnecting = false;
			});
			console.time('scrambo.initialize took');
			scrambo.initialize(Math);
			console.timeEnd('scrambo.initialize took');
		},
		onGiikerMove(move) {
			const now = new Date();

			if (this.phase === 'prescramble') {
				if (GiiKER.cube.isSolved()) {
					this.phase = 'scramble';
				}
				return;
			}

			if (this.phase === 'scramble') {
				this.scramble.unshift({
					face: move.face,
					amount: -move.amount,
				});
				if (this.scramble.length > this.placeholderMoves.length) {
					this.placeholderMoves = this.scramble.moves.map((m) => ({...m}));
				}
				if (this.scramble.length === 0) {
					this.phase = 'inspect';
					this.time = 0;
				}
				return;
			}

			if (this.phase === 'inspect') {
				this.startTime = new Date();
				this.cross = null;
				this.cll = null;
				this.pll = null;
				this.oll = null;
				this.rouxBlock = null;
				this.phase = 'solve';
				this.isFirstSolve = false;
				this.isDescriptionShown = false;
				this.cubeStage = 'unknown';
				this.stages = Object.assign(
					...config.stagesData.cfop.map(({id}) => ({
						[id]: {
							sequence: new MoveSequence(),
							time: null,
						},
					})),
					...config.stagesData.roux.map(({id}) => ({
						[id]: {
							sequence: new MoveSequence(),
							time: null,
						},
					})),
				);
				this.isOll2Look = false;
				this.pllLooks = [];
				this.interval = setInterval(this.onTick, 1000 / 30);
				this.scrollToStage();
				// fall through
			}

			if (this.phase === 'solve') {
				this.time = now.getTime() - this.startTime.getTime();
				this.turns.push({time: this.time, ...move});

				this.stages[this.cubeStage].sequence.push({time: this.time, ...move});

				if (this.cubeStage === 'unknown') {
					const cross = findCross(GiiKER.cube);
					const rouxBlock = findRouxBlock(GiiKER.cube);
					if (cross) {
						this.mode = 'cfop';
						this.cubeStage = 'f2l1';
						this.scrollToStage();
						this.stages.unknown.time = this.time;
						this.cross = cross;
						// fall through
					} else if (rouxBlock) {
						this.mode = 'roux';
						this.cubeStage = 'block2';
						this.scrollToStage();
						this.stages.unknown.time = this.time;
						this.rouxBlock = rouxBlock;
						// fall through
					}
				}

				for (const stage of config.stagesData[this.mode].slice(1)) {
					if (this.cubeStage === stage.id) {
						const {result, oll, pll, cll} = isStageSatisfied({
							mode: this.mode,
							cube: GiiKER.cube,
							stage: stage.id,
							cross: this.cross,
							rouxBlock: this.rouxBlock,
						});

						if (result === true) {
							this.cubeStage = getNextStage(stage.id);
							this.scrollToStage();
							this.stages[stage.id].time = this.time;
							if (stage.id === 'f2l4') {
								this.oll = oll;
							}
							if (stage.id === 'oll') {
								this.pll = pll;
							}
							if (stage.id === 'block2') {
								this.cll = cll;
							}
						} else if (stage.id === 'oll' && !this.oll.isEdgeOriented && oll !== undefined && oll.isEdgeOriented) {
							this.isOll2Look = true;
						}

						if (pll !== undefined && pll.name !== 'PLL Skip') {
							this.pllLooks = uniq([...this.pllLooks, pll.name]);
						}
					}
				}

				if (GiiKER.cube.isSolved()) {
					this.finishSolve({isError: false});
				}
			}
		},
		onTick() {
			const now = new Date();
			this.time = now.getTime() - this.startTime.getTime();
		},
		scrollToStage() {
			const element = document.getElementById(this.cubeStage);
			if (element) {
				element.scrollIntoView({block: 'end', inline: 'nearest', behavior: 'smooth'});
			}
		},
		onClickReset() {
			if (this.phase !== 'solve') {
				return;
			}

			this.finishSolve({isError: true});
			GiiKER.cube.identity();
		},
		async finishSolve({isError}) {
			clearInterval(this.interval);

			const {inspection: ollInspection} = (this.stages.oll.time !== null && this.stages.oll.sequence.length !== 0)
				? getInspectionTime({stage: this.stages.oll, cross: this.cross, previousTime: this.stages.f2l4.time})
				: {inspection: null};

			const {inspection: pllInspection} = (this.stages.pll.time !== null && this.stages.pll.sequence.length !== 0)
				? getInspectionTime({stage: this.stages.pll, cross: this.cross, previousTime: this.stages.oll.time})
				: {inspection: null};

			const {solve} = await saveSolve({
				mode: this.mode,
				date: this.startTime.getTime(),
				time: this.time,
				scramble: this.initialScramble.moves,
				turns: this.turns.moves,
				stages: this.getSerializedStages(),
				isError,
				moveCount: this.moveCount,
				crossFace: this.cross ? this.cross : null,
				isXcross: this.isXcross,
				ollCase: this.oll ? this.oll.index : null,
				pllCase: this.pll ? this.pll.index : null,
				ollLooks: this.oll ? (this.isOll2Look ? 2 : 1) : null,
				pllLooks: this.pll ? this.pllLooks.length : null,
				ollInspection,
				pllInspection,
				cllCase: this.cll ? this.cll.index : null,
				rouxBlockSide: this.rouxBlock ? this.rouxBlock.side : null,
				rouxBlockBottom: this.rouxBlock ? this.rouxBlock.bottom : null,
			});

			this.previousSolve = solve;
			this.phase = 'scramble';
			this.isFirstSolve = false;
			const scramble = scrambo.getRandomScramble().replace(/ +/g, ' ').trim();
			this.scramble = MoveSequence.fromScramble(scramble, {mode: 'reduction'});
			this.initialScramble = MoveSequence.fromScramble(scramble, {mode: 'reduction'});
			this.turns = new MoveSequence([], {mode: 'raw'});
			this.placeholderMoves = this.scramble.moves.map((move) => ({...move}));
			document.getElementById('stages').scrollTop = 0;
		},
		getSerializedStages() {
			return config.stagesData[this.mode].map(({id}, index, stagesData) => {
				const stage = this.stages[id];

				if (!stage) {
					return undefined;
				}

				const previousStage = index === 0 ? undefined : this.stages[stagesData[index - 1].id];
				const time = stage.time - (previousStage ? previousStage.time : 0);

				if (!this.cross) {
					return {
						id,
						time,
						turns: stage.sequence.toObject(),
					};
				}

				const rotation = getRotation({from: this.cross, to: 'D'});
				const turns = stage.sequence.toObject({cross: this.cross});
				if (id === 'unknown' && rotation.amount !== 0) {
					turns.unshift(rotation);
				}

				return {
					id,
					time,
					turns,
				};
			}).filter((stage) => stage);
		},
		async onClickDelete() {
			if (this.previousSolve === null) {
				return;
			}

			await deleteSolve(this.previousSolve);
			this.previousSolve = null;
			this.isDeleteDialogOpen = false;
		},
	},
	head() {
		return {
			title: 'Smart Cube Timer',
		};
	},
};
</script>

<style>
	.controls {
		text-align: center;
		flex: 0 0 auto;

		.scramble {
			max-width: 110vmin;
			font-size: 8vmin;
			font-weight: bold;
			line-height: 1.2em;
			margin: 0 auto;
		}

		.timer {
			font-size: 20vmin;
			font-weight: bold;
			line-height: 1em;
			display: flex;
			justify-content: center;
			align-items: center;
		}

		.timer .v-dialog__activator {
			display: flex;
		}

		.solve-info {
			margin: 0 0.2rem;
		}
	}

	.times {
		flex: 1 1 0;
		padding-top: 0 !important;
		overflow-y: auto;
	}
</style>
