import express from "express";
import multer from "multer";
import path from "path";
import createError from "http-errors";

import { uploadFileDiver } from "../controllers/fileController";
import { verifyToken } from "../middleware/verify-token";

const storage = multer.diskStorage({
	destination: function (req, file, cb) {
		cb(null, `${path.join(__dirname, "../assets/uploads/")}`);
	},
	filename: function (req, file, cb) {
		cb(null, file.fieldname + "_" + Date.now() + "_" + file.originalname);
	},
});

const upload = multer({
	storage,
	fileFilter: (req, file, cb) => {
		if (!["image/jpg", "image/png", "image/jpeg"].includes(file.mimetype)) {
			return cb(new createError.InternalServerError("Incorrect file format"));
		}
		cb(null, true);
	},
});

const route = express.Router();

route.post(
	"/upload-image",
	verifyToken,
	upload.single("file"),
	uploadFileDiver
);

export default route;
