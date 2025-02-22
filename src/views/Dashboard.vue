<template>
	<v-container class="mt-5">
		<v-row align="center" justify="center">
			<v-col cols="12" sm="12" md="8">
				<div class="purple--text text-h3 font-weight-bold">Dashboard</div>
				<v-layout align-center>
					<v-icon x-large color="black" class="mt-5">mdi-account</v-icon>
					<div class="mt-5 text-h5">Welcome {{ user.displayName }}!</div>
				</v-layout>
				<v-layout wrap>
					<v-btn
						class="mt-5 mr-5 px-5 no-active"
						color="primary"
						:to="{ name: 'EditProfile' }"
					>
						<v-icon left>mdi-account-circle</v-icon>
						Edit Profile
					</v-btn>
					<v-btn
						class="mt-5 mr-5 px-5 no-active"
						color="primary"
						link
						:to="{ name: 'Contracts' }"
					>
						<v-icon left>mdi-account-tie</v-icon>
						View Contracts
					</v-btn>
					<v-btn
						@click="deleteAccount"
						class="mt-5 mr-5 px-5 py-xs-5"
						color="error"
					>
						<v-icon left>mdi-account-minus</v-icon>
						Delete my account
					</v-btn>
				</v-layout>
			</v-col>
			<v-col cols="6" md="2">
				<v-layout column>
					<img :src="user.photoURL" />
					<v-dialog max-width="600px" class="elevation-0">
						<template v-slot:activator="{ on, attrs }">
							<v-btn tile v-bind="attrs" v-on="on" color="primary"
								><v-icon>mdi-image-edit</v-icon></v-btn
							>
						</template>
						<v-card class="pb-5">
							<v-alert :type="alert.type" v-if="alert.msg">{{
								alert.msg
							}}</v-alert>
							<v-card-title class="text-h5 grey lighten-2">
								Change your profile picutre
							</v-card-title>
							<v-form class="px-3">
								<v-file-input
									class="my-5"
									prepend-icon=""
									prepend-inner-icon="mdi-camera"
									label="New profile picture"
									v-model="selectedImage"
									@change="previewImage"
								></v-file-input>
							</v-form>
							<v-img
								max-width="400px"
								class="mx-auto rounded-lg mb-2"
								style="border: 2px solid black;"
								v-if="selectedImage"
								:src="selectedImageURL"
							></v-img>
							<v-layout>
								<v-btn
									:disabled="isUploadingImage ? true : false"
									color="primary"
									class="mx-auto"
									@click="setNewProfilePicture"
									>Set as new profile picture</v-btn
								>
							</v-layout>
						</v-card>
					</v-dialog>
				</v-layout>
			</v-col>
		</v-row>
		<v-row align="center" justify="center" class="mb-15">
			<v-col cols="12" sm="12" md="10">
				<div class="my-5 text-h5 font-weight-bold">Your listings</div>
				<v-simple-table>
					<thead>
						<tr>
							<th class="text-left">Product Name</th>
							<th class="text-left">Product Description</th>
							<th class="text-left">Rented</th>
							<th></th>
							<th></th>
						</tr>
					</thead>
					<tbody>
						<tr v-for="listing in listings" :key="listing.id">
							<td>{{ listing.title }}</td>
							<td>
								{{ listing.description }}
							</td>
							<td>No</td>
							<td>
								<v-btn icon color="error">
									<v-icon>mdi-delete</v-icon>
								</v-btn>
							</td>
							<td>
								<v-btn icon color="warning">
									<v-icon>mdi-pen</v-icon>
								</v-btn>
							</td>
						</tr>
					</tbody>
				</v-simple-table>
			</v-col>
		</v-row>
	</v-container>
</template>

<script>
import {
	auth,
	getDoc,
	db,
	doc,
	collection,
	query,
	where,
	storage,
	updateProfile,
	ref,
	uploadBytes,
	getDownloadURL,
	deleteObject,
	getDocs,
} from "@/firebase";
import { v4 as uuid } from "uuid";
import { mapGetters, mapMutations } from "vuex";

export default {
	name: "Dashboard",
	data() {
		return {
			selectedImage: null,
			selectedImageURL: "",
			isUploadingImage: false,
			listings: [],
		};
	},
	mounted() {
		this.getMyProducts();
	},
	computed: {
		...mapGetters({ user: "user", alert: "alert" }),
	},
	methods: {
		...mapMutations({ notify: "notify" }),
		async getMyProducts() {
			try {
				const productsRef = collection(db, "products");

				const q = query(productsRef, where("author.uid", "==", this.user.uid));

				const querySnapshot = await getDocs(q);

				let listings = [];

				querySnapshot.forEach((doc) => {
					listings.push({ id: doc.id, ...doc.data() });
				});

				this.listings = listings;
			} catch (error) {
				console.log(error);
			}
		},
		async deleteAccount() {
			const currentUser = auth.currentUser;
			const userID = currentUser.uid;
			// await deleteUser(currentUser);
			// await deleteDoc(doc(db, 'users', userID));
			// await deleteDoc(doc(db, 'products'))

			const userRef = await doc(db, "users", userID);
			const user = await getDoc(userRef);
			const userProductsRef = await query(
				collection(db, "products"),
				where("author.uid" == userID)
			);

			// userProducts.forEach(doc => {
			//    console.log(doc.data())
			// });
			console.log(user.data());
		},
		async setNewProfilePicture() {
			const currentUser = auth.currentUser;
			const userID = currentUser.uid;

			try {
				const exists = await getDownloadURL(ref(storage, currentUser.photoURL));

				const oldImage = ref(storage, currentUser.photoURL);

				console.log(oldImage);
				console.log("I'm here");
				await deleteObject(oldImage);
			} catch (error) {
				console.log("");
			}

			if (this.selectedImage) {
				const imageName = uuid();
				const storageRef = ref(
					storage,
					`users/${userID}/profilePic/${imageName}`
				);
				this.isUploadingImage = true;
				await uploadBytes(storageRef, this.selectedImage);
				const downloadURL = await getDownloadURL(storageRef);

				await updateProfile(currentUser, {
					photoURL: downloadURL,
				});
				this.notify({
					type: "success",
					msg: "Successfully changed the profile picture",
				});
				this.isUploadingImage = false;
			}
		},
		previewImage() {
			if (this.selectedImage != null) {
				this.selectedImageURL = URL.createObjectURL(this.selectedImage);
			} else {
				this.selectedImageURL = "";
			}
		},
	},
};
</script>

<style></style>
