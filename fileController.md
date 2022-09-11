import { Response, Request, NextFunction } from "express";
import { google } from "googleapis";
import fs from "fs";
import createError from "http-errors";

const oauth2Client = new google.auth.OAuth2(
	process.env.CLIENT_ID_GOOGLE_CLOUD,
	process.env.CLIENT_SECRET_GOOGLE_CLOUD,
	process.env.REDIRECT_URI_AUTH_PLAYGROUND
);
oauth2Client.setCredentials({
	refresh_token: process.env.REFRESH_TOKEN_GOOGLE_PLAYGROUND,
});

const driver = google.drive({ version: "v3", auth: oauth2Client });

const deleteFile = (filePath: string) => {
	fs.unlink(filePath, (err) => {
		if (err) {
			throw new createError.NotFound("File does not exist");
		}
	});
};

const filePublic = async (fileId: string | null | undefined) => {
	try {
		if (!fileId) {
			throw new createError.InternalServerError("File does not exist");
		}

		await driver.permissions.create({
			fileId,
			requestBody: {
				role: "reader",
				type: "anyone",
			},
		});

		const getUrl = await driver.files.get({
			fileId,
			fields: "webViewLink, webContentLink",
		});
		return getUrl;
	} catch (error: any) {
		throw new createError.InternalServerError(error.message);
	}
};

export const uploadFileDiver = async (
	req: Request,
	res: Response,
	next: NextFunction
) => {
	try {
		if (!req.file) {
			return next(new createError.NotFound("File does not exist"));
		}

		const fileData = await driver.files.create({
			requestBody: {
				name: req.file.originalname,
				parents: ["1KNThy7WWCzS4xVrhvOag2KJr12CXyl81"],
			},
			media: {
				mimeType: req.file.mimetype,
				body: fs.createReadStream(req.file.path),
			},
		});

		deleteFile(req.file.path);

		const getUrl = await filePublic(fileData.data.id);

		return res
			.status(200)
			.json({ data: { id: fileData.data.id, ...getUrl.data } });
	} catch (error: any) {
		return next(new createError.InternalServerError(error.message));
	}
};
