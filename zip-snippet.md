
```java
import java.io.*;
import java.nio.file.*;
import java.util.zip.*;

public class ZipCsvFiles {

    public static void main(String[] args) throws IOException {
        // 1️⃣ Folder where your CSV files are
        String folderPath = "C:\\path\\to\\your\\csv\\directory";

        // 2️⃣ List CSV files
        Files.list(Paths.get(folderPath))
                .filter(path -> path.toString().toLowerCase().endsWith(".csv"))
                .forEach(ZipCsvFiles::zipFileWithoutExtension);
    }

    private static void zipFileWithoutExtension(Path csvPath) {
        try {
            // Original CSV file name (without extension)
            String fileNameWithoutExt = csvPath.getFileName().toString().replaceFirst("[.][^.]+$", "");

            // Output ZIP file (no extension)
            Path zipPath = csvPath.getParent().resolve(fileNameWithoutExt);

            // Create ZIP output stream
            try (FileOutputStream fos = new FileOutputStream(zipPath.toFile());
                 ZipOutputStream zos = new ZipOutputStream(fos)) {

                // Add CSV to ZIP
                ZipEntry zipEntry = new ZipEntry(csvPath.getFileName().toString());
                zos.putNextEntry(zipEntry);

                // Copy file data into ZIP
                Files.copy(csvPath, zos);

                zos.closeEntry();
            }

            System.out.println("Zipped: " + csvPath + " -> " + zipPath);
        } catch (IOException e) {
            System.err.println("Error zipping file: " + csvPath);
            e.printStackTrace();
        }
    }
}
```