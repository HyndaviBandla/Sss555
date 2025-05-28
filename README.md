import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;

import java.io.*;
import java.nio.file.*;
import java.util.*;

public class EventFileOrganizer {

    public static void main(String[] args) {
        File excelFile = new File("Event_data.xlsx");
        File workingDir = excelFile.getParentFile();
        File eqOtcFolder = new File(workingDir, "EQ_OTC");

        if (!eqOtcFolder.exists()) {
            eqOtcFolder.mkdir();
        }

        try (FileInputStream fis = new FileInputStream(excelFile);
             Workbook workbook = new XSSFWorkbook(fis)) {

            Sheet sheet = workbook.getSheet("EQ_OTC");
            if (sheet == null) {
                System.out.println("Sheet EQ_OTC not found!");
                return;
            }

            Map<String, List<RowData>> uuidMap = new HashMap<>();

            // Assume header is in row 0
            int uuidCol = getColumnIndex(sheet, "uuid_regu_realtime_traderef.keyword");
            int eventTypeCol = getColumnIndex(sheet, "event_type_regu_realtime_traderef.keyword");
            int eventIdCol = getColumnIndex(sheet, "id_regu_realtime_event");

            for (Row row : sheet) {
                if (row.getRowNum() == 0) continue; // skip header

                String uuid = getCellValue(row.getCell(uuidCol));
                String eventType = getCellValue(row.getCell(eventTypeCol));
                String eventId = getCellValue(row.getCell(eventIdCol));

                if (uuid == null || eventType == null || eventId == null) continue;

                uuidMap.computeIfAbsent(uuid, k -> new ArrayList<>())
                       .add(new RowData(eventType, eventId));
            }

            for (String uuid : uuidMap.keySet()) {
                File uuidFolder = new File(eqOtcFolder, uuid);
                if (!uuidFolder.exists()) uuidFolder.mkdir();

                List<RowData> rows = uuidMap.get(uuid);
                for (RowData row : rows) {
                    File eventTypeFolder = new File(uuidFolder, row.eventType);
                    if (!eventTypeFolder.exists()) eventTypeFolder.mkdir();

                    // Look for files matching eventId
                    String[] patterns = {
                        row.eventId + "_out.zip",
                        row.eventId + "_in.pdl",
                        row.eventId + "_out.pdl"
                    };

                    for (String pattern : patterns) {
                        File sourceFile = new File(workingDir, pattern);
                        if (sourceFile.exists()) {
                            File targetFile = new File(eventTypeFolder, sourceFile.getName());
                            Files.copy(sourceFile.toPath(), targetFile.toPath(),
                                       StandardCopyOption.REPLACE_EXISTING);
                            System.out.println("Copied " + sourceFile.getName() + " → " + targetFile.getAbsolutePath());
                        }
                    }
                }
            }

            System.out.println("Finished organizing files!");

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static int getColumnIndex(Sheet sheet, String header) {
        Row headerRow = sheet.getRow(0);
        for (Cell cell : headerRow) {
            if (header.equalsIgnoreCase(cell.getStringCellValue())) {
                return cell.getColumnIndex();
            }
        }
        throw new RuntimeException("Header not found: " + header);
    }

    private static String getCellValue(Cell cell) {
        if (cell == null) return null;
        switch (cell.getCellType()) {
            case STRING: return cell.getStringCellValue();
            case NUMERIC: return String.valueOf((long) cell.getNumericCellValue());
            default: return null;
        }
    }

    static class RowData {
        String eventType;
        String eventId;
        RowData(String eventType, String eventId) {
            this.eventType = eventType;
            this.eventId = eventId;
        }
    }
}
