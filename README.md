import unittest
import os
import shutil
from unittest.mock import patch, mock_open, MagicMock
from datetime import datetime
import re
from Entities.Customentities import ApihomeResp
from Services import CustomException
import fileoperations as fops
import globalvars as gvar

class TestFileOperations(unittest.TestCase):

    def setUp(self):
        self.test_folder = "test_folder"
        self.input_folder = os.path.join(self.test_folder, "input")
        os.makedirs(self.input_folder, exist_ok=True)
        gvar.sadrd_settings = []
        gvar.user_id = "test_user"
        fops.dbops_obj = MagicMock() # Mock the dboperations object

    def tearDown(self):
        shutil.rmtree(self.test_folder, ignore_errors=True)

    def create_dummy_file(self, filename, content=""):
        filepath = os.path.join(self.input_folder, filename)
        with open(filepath, "w") as f:
            f.write(content)
        return filepath

    def test_ExtractFilesToLoad_valid_files(self):
        self.create_dummy_file("test1.xlsx")
        self.create_dummy_file("test2.xlsm")
        self.create_dummy_file("test3.csv")
        self.create_dummy_file("ignore.txt")

        expected_files = [
            os.path.join(self.input_folder, "test1.xlsx"),
            os.path.join(self.input_folder, "test2.xlsm"),
            os.path.join(self.input_folder, "test3.csv"),
        ]
        actual_files = fops.ExtractFilesToLoad(self.test_folder, "some_action")
        self.assertEqual(sorted(actual_files), sorted(expected_files))

    def test_ExtractFilesToLoad_no_valid_files(self):
        self.create_dummy_file("ignore1.txt")
        self.create_dummy_file("ignore2")

        actual_files = fops.ExtractFilesToLoad(self.test_folder, "some_action")
        self.assertEqual(actual_files, [])

    def test_getinpfilenames_toprocess_valid_files(self):
        self.create_dummy_file("data1.xlsx")
        self.create_dummy_file("data2.xlsm")
        self.create_dummy_file("data3.csv")
        self.create_dummy_file("temp.txt")

        expected_files = [
            os.path.join(self.input_folder, "data1.xlsx"),
            os.path.join(self.input_folder, "data2.xlsm"),
            os.path.join(self.input_folder, "data3.csv"),
        ]
        actual_files = fops.getinpfilenames_toprocess(self.test_folder, "")
        self.assertEqual(sorted(actual_files), sorted(expected_files))

    def test_getinpfilenames_toprocess_no_valid_files(self):
        self.create_dummy_file("log.txt")
        self.create_dummy_file("backup")

        actual_files = fops.getinpfilenames_toprocess(self.test_folder, "")
        self.assertEqual(actual_files, [])

    @patch("os.makedirs")
    @patch("os.listdir")
    @patch("shutil.copyfile")
    def test_DownloadServerFilesToLoad_annual_schd_success(self, mock_copyfile, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["2025JHUSASchDPart1.csv", "ignore.txt"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD'),
                               MagicMock(settingName='Valid_Company', settingValue='JHUSA')]

        result = fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertEqual(len(result), 1)
        self.assertTrue(os.path.join(self.input_folder, "2025JHUSASchDPart1.csv") in result)
        mock_copyfile.assert_called_once_with("server_path//2025JHUSASchDPart1.csv", os.path.join(self.input_folder, "2025JHUSASchDPart1.csv"))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    @patch("shutil.copyfile")
    def test_DownloadServerFilesToLoad_qualpctftc_success(self, mock_copyfile, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["data_QualPct.xlsx", "other.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_QualFTC', settingValue='QualPct')]

        result = fops.DownloadServerFilesToLoad("server_path", self.input_folder, "QualPctFTC", "2025")
        self.assertEqual(len(result), 1)
        self.assertTrue(os.path.join(self.input_folder, "data_QualPct.xlsx") in result)
        mock_copyfile.assert_called_once_with("server_path//data_QualPct.xlsx", os.path.join(self.input_folder, "data_QualPct.xlsx"))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    @patch("shutil.copyfile")
    def test_DownloadServerFilesToLoad_ftcgrossup_success(self, mock_copyfile, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["FTCGrossup_Q1.xlsm", "FTCGrossup_Q2.xls", "FTCGrossup_Q3.xlsx", "FTCGrossup_Q4.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_FTCGrossup', settingValue='FTCGrossup'),
                               MagicMock(settingName='Valid_Quarter', settingValue='Q1'),
                               MagicMock(settingName='Valid_Quarter', settingValue='Q2'),
                               MagicMock(settingName='Valid_Quarter', settingValue='Q3'),
                               MagicMock(settingName='Valid_Quarter', settingValue='Q4')]

        result = fops.DownloadServerFilesToLoad("server_path", self.input_folder, "FTCGrossup", "2025")
        self.assertEqual(len(result), 4)
        expected_files = [
            os.path.join(self.input_folder, "FTCGrossup_Q1.xlsm"),
            os.path.join(self.input_folder, "FTCGrossup_Q2.xls"),
            os.path.join(self.input_folder, "FTCGrossup_Q3.xlsx"),
            os.path.join(self.input_folder, "FTCGrossup_Q4.csv"),
        ]
        self.assertEqual(sorted(result), sorted(expected_files))
        self.assertEqual(mock_copyfile.call_count, 4)
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_no_files_found(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = []
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: No files found."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertEqual(str(context.exception), "Error: No files found.")
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_file_type_annual(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["test.xlsx"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid file type(s) found: test.xlsx. Expected .csv."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertTrue("Invalid file type(s) found: test.xlsx. Expected .csv" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_file_type_qualpct(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["data_QualPct.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_QualFTC', settingValue='QualPct')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid file type(s) found: data_QualPct.csv. Expected .xlsx, .xls, or .xlsm."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "QualPctFTC", "2025")
        self.assertTrue("Invalid file type(s) found: data_QualPct.csv. Expected .xlsx, .xls, or .xlsm" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_file_type_ftcgrossup(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["FTCGrossup_Q1.txt"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_FTCGrossup', settingValue='FTCGrossup')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid file type(s) found: FTCGrossup_Q1.txt. Expected .xlsx, .xls, or .xlsm."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "FTCGrossup", "2025")
        self.assertTrue("Invalid file type(s) found: FTCGrossup_Q1.txt. Expected .xlsx, .xls, or .xlsm" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_filename_format_annual(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["2025InvalidSchD.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD'),
                               MagicMock(settingName='Valid_Company', settingValue='JHUSA')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid filename format: 2025InvalidSchD.csv. Expected filename to contain 'SchD'."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertTrue("Invalid filename format: 2025InvalidSchD.csv. Expected filename to contain 'SchD'" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_filename_format_qualpct(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["dataInvalid.xlsx"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_QualFTC', settingValue='QualPct')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid filename format: dataInvalid.xlsx. Expected filename to contain 'QualPct'."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "QualPctFTC", "2025")
        self.assertTrue("Invalid filename format: dataInvalid.xlsx. Expected filename to contain 'QualPct'" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_filename_format_ftcgrossup(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["Invalid_Q1.xlsx"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_FTCGrossup', settingValue='FTCGrossup')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid filename format: Invalid_Q1.xlsx. Expected filename to contain 'FTCGrossup'."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "FTCGrossup", "2025")
        self.assertTrue("Invalid filename format: Invalid_Q1.xlsx. Expected filename to contain 'FTCGrossup'" in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_file_open_error(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["test.xlsx"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD')]
        fops.is_open = MagicMock(return_value=True)
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: File is currently open: test.xlsx."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertTrue("File is currently open: test.xlsx." in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_incorrect_year(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["2024TestSchD.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD')]
        fops.dbops_obj.BuildErrorMessage.return_value = "Error: Invalid year in filename: 2024TestSchD.csv. Expected year: 2025."

        with self.assertRaises(CustomException.FileValidationException) as context:
            fops.DownloadServerFilesToLoad("server_path", self.input_folder, "Annual Stmt - Sch D", "2025")
        self.assertTrue("Invalid year in filename: 2024TestSchD.csv. Expected year: 2025." in str(context.exception))
        fops.dbops_obj.insert_actionLog.assert_called_once()

    @patch("os.makedirs")
    @patch("os.listdir")
    def test_DownloadServerFilesToLoad_invalid_company(self, mock_listdir, mock_makedirs):
        mock_listdir.return_value = ["2025InvalidCompanySchD.csv"]
        gvar.sadrd_settings = [MagicMock(settingName='Filename_AnnStmtSchD', settingValue='SchD'),
                               MagicMock(settingName='Valid_Company', settingValue='JHUSA
